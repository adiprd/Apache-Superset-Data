# Apache Superset - Data Access and Dataset Configuration Guide

## Table of Contents
- [Dataset Setup and Configuration](#dataset-setup-and-configuration)
- [Data Access Control](#data-access-control)
- [Row Level Security](#row-level-security)
- [Database Connections](#database-connections)
- [SQL Lab Configuration](#sql-lab-configuration)
- [Data Refresh and Caching](#data-refresh-and-caching)
- [Troubleshooting](#troubleshooting)

## Dataset Setup and Configuration

### Adding Database Connection

#### Through UI:
1. Navigate to `Sources` → `Databases`
2. Click `+ DATABASE`
3. Select PostgreSQL
4. Configure connection parameters:

```
Database: PostgreSQL
Host: your-postgres-host
Port: 5432
Database Name: your_database
Username: superset_user
Password: ********
```

#### Through CLI:
```bash
# Export database URI
export DATABASE_URI="postgresql://superset_user:password@host:5432/your_database"

# Or using superset CLI
superset set-database-uri \
    --database-name "Production DB" \
    --uri postgresql://superset_user:password@host:5432/your_database
```

### Advanced Database Configuration

```python
# In superset_config.py
PREVENT_UNSAFE_DB_CONNECTIONS = False

# Additional connection parameters
ADDITIONAL_DATABASE_ENGINE_PARAMETERS = {
    'postgresql': {
        'connect_args': {
            'sslmode': 'require',
            'application_name': 'superset'
        }
    }
}
```

## Data Access Control

### User Roles and Permissions

#### Default Roles in Superset:
- **Admin**: Full access
- **Alpha**: Access to all data sources
- **Gamma**: Limited access (need specific permissions)
- **sql_lab**: SQL Lab access only
- **Public**: No access by default

### Creating Custom Role

```python
# In superset_config.py
from superset.security import SupersetSecurityManager

class CustomSecurityManager(SupersetSecurityManager):
    def __init__(self, appbuilder):
        super(CustomSecurityManager, self).__init__(appbuilder)
        
        # Define custom roles
        self.define_custom_roles()

    def define_custom_roles(self):
        self.appbuilder.sm.add_role("Data Analyst")
        self.appbuilder.sm.add_role("Sales Manager")
        self.appbuilder.sm.add_role("Branch Viewer")
```

### Assigning Permissions

#### Through UI:
1. Navigate to `Security` → `List Roles`
2. Select role to configure
3. Set permissions:

**For Data Analyst:**
- `can explore on Superset`
- `can explore json on Superset`
- `can csv on Superset`
- `can share dashboard`
- `can share chart`

**For Sales Manager:**
- `can dashboard on Superset`
- `can explore on Superset`
- `can list dashboard`
- `can chart on Superset`

## Row Level Security (RLS)

### Setup RLS Rules

#### Through UI:
1. Navigate to `Security` → `Row Level Security`
2. Click `+ ADD RULE`
3. Configure rule parameters:

```python
# Example RLS Rule for Branch-based access
Rule Name: "Branch Access - Jakarta"
Table: "data_transaksi_sparepart"
Clause: "cabang = 'JAKARTA'"
Roles: ["Sales Jakarta", "Gamma"]

Rule Name: "Branch Access - Bandung" 
Table: "data_transaksi_sparepart"
Clause: "cabang = 'BANDUNG'"
Roles: ["Sales Bandung", "Gamma"]
```

#### Through API:
```python
import requests

def create_rls_rule(rule_data):
    headers = {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
    }
    
    response = requests.post(
        'http://superset-host/api/v1/rowlevelsecurity/',
        headers=headers,
        json=rule_data
    )
    return response.json()

# Example rule data
rule_data = {
    "name": "Branch Access - Surabaya",
    "tables": [{"table_name": "data_transaksi_sparepart"}],
    "filter_type": "Regular",
    "clause": "cabang = 'SURABAYA'",
    "roles": [{"id": 3}]  # Role ID for Sales Surabaya
}
```

### Dynamic RLS with Jinja Templates

```sql
-- Using Jinja for dynamic user context
SELECT *
FROM mb.data_transaksi_sparepart
WHERE cabang IN (
    {% for branch in current_user.branches %}
        '{{ branch }}'{% if not loop.last %},{% endif %}
    {% endfor %}
)

-- Or using macro
SELECT *
FROM mb.data_transaksi_sparepart
WHERE cabang = '{{ current_user.get_username().upper() }}'
```

### RLS for Complex Scenarios

```python
# In superset_config.py
from flask import g
import json

def get_user_branches():
    """Dynamic function to get user branches"""
    user_branches = {
        'user_jakarta': ['JAKARTA'],
        'user_bandung': ['BANDUNG'], 
        'user_regional': ['JAKARTA', 'BANDUNG', 'SURABAYA']
    }
    return user_branches.get(g.user.username, [])

# Register function for use in Jinja
JINJA_CONTEXT_ADDONS = {
    'user_branches': get_user_branches
}
```

## Database Connections Management

### Multiple Database Connections

```python
# Configure multiple databases
DATABASES = {
    'production': {
        'sqlalchemy_uri': 'postgresql://user:pass@prod-host:5432/prod_db',
        'cache_timeout': 300,
        'metadata_params': {},
        'engine_params': {
            'pool_size': 10,
            'max_overflow': 20,
            'pool_pre_ping': True
        }
    },
    'staging': {
        'sqlalchemy_uri': 'postgresql://user:pass@staging-host:5432/staging_db',
        'cache_timeout': 600,
        'engine_params': {
            'pool_size': 5,
            'max_overflow': 10
        }
    }
}
```

### Connection Pool Configuration

```python
# In superset_config.py
SQLALCHEMY_DATABASE_URI = 'postgresql://superset:password@localhost/superset'
SQLALCHEMY_ENGINE_OPTIONS = {
    'pool_size': 10,
    'max_overflow': 20,
    'pool_recycle': 3600,
    'pool_pre_ping': True
}

# For source databases
DATA_DB_ENGINE_PARAMS = {
    'pool_size': 20,
    'max_overflow': 30,
    'pool_recycle': 1800,
    'pool_pre_ping': True
}
```

## SQL Lab Configuration

### SQL Lab Settings

```python
# In superset_config.py
FEATURE_FLAGS = {
    'ENABLE_TEMPLATE_PROCESSING': True,
    'SQL_VALIDATORS_BY_ENGINE': {
        'postgresql': 'PostgreSQLValidator',
    }
}

# SQL Lab limits
SQLLAB_CTAS_NO_LIMIT = True
SQLLAB_TIMEOUT = 3600  # 1 hour
SQLLAB_VALIDATION_TIMEOUT = 300  # 5 minutes
SQLLAB_ASYNC_TIME_LIMIT_SEC = 3600

# Query results limits
DISPLAY_SQL_MAX_ROW = 100000
SQL_MAX_ROW = 100000
```

### Template Variables for SQL Lab

```sql
-- Using template variables
SELECT *
FROM mb.data_transaksi_sparepart
WHERE tgl_invoice BETWEEN '{{ from_dttm }}' AND '{{ to_dttm }}'
  AND cabang IN ({{ "'" + "','".join(filter_values('cabang')) + "'" }})
  AND total_harga > {{ numeric_filter('min_price') }}

-- Dynamic WHERE clause
SELECT *
FROM mb.data_transaksi_sparepart
WHERE 1=1
{% if filter_values('cabang') %}
  AND cabang IN ({{ "'" + "','".join(filter_values('cabang')) + "'" }})
{% endif %}
{% if from_dttm is defined %}
  AND tgl_invoice >= '{{ from_dttm }}'
{% endif %}
```

## Data Refresh and Caching

### Dataset Refresh Configuration

```python
# Cache configuration
CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0'
}

DATA_CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 86400,
    'CACHE_KEY_PREFIX': 'data_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/1'
}

# Celery configuration for async queries
class CeleryConfig(object):
    broker_url = 'redis://localhost:6379/0'
    imports = ('superset.sql_lab',)
    result_backend = 'redis://localhost:6379/0'
    worker_prefetch_multiplier = 1
    task_acks_late = False

CELERY_CONFIG = CeleryConfig
```

### Automated Data Refresh

```python
# Scheduled cache warming
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    'cache-warmup-daily': {
        'task': 'superset.tasks.cache_warmup',
        'schedule': crontab(hour=0, minute=30),
        'kwargs': {
            'strategy_name': 'top_n_dashboards',
            'top_n': 5,
            'since': '7 days ago',
        },
    },
    'refresh-datasets': {
        'task': 'superset.tasks.refresh_datasources',
        'schedule': crontab(hour=1, minute=0),
    }
}
```

## Practical Examples

### Complete Dataset Setup for Sparepart Data

```python
# Script for initial datasets setup
def setup_sparepart_datasets():
    """Setup all required datasets"""
    
    datasets = [
        {
            'name': 'Sparepart Transactions',
            'table_name': 'data_transaksi_sparepart',
            'schema': 'mb',
            'metrics': [
                {
                    'metric_name': 'total_penjualan',
                    'expression': 'SUM(total_harga)'
                },
                {
                    'metric_name': 'total_transaksi', 
                    'expression': 'COUNT(DISTINCT no_invoice)'
                },
                {
                    'metric_name': 'avg_order_value',
                    'expression': 'AVG(total_harga)'
                }
            ],
            'columns': [
                'cabang', 'tgl_invoice', 'no_invoice', 'kode_barang',
                'nama_barang', 'tipe_sparepart', 'total_harga', 'qty'
            ]
        },
        {
            'name': 'Product Performance',
            'table_name': 'vw_product_performance', 
            'schema': 'mb',
            'metrics': [
                {
                    'metric_name': 'revenue',
                    'expression': 'SUM(revenue)'
                },
                {
                    'metric_name': 'quantity_sold',
                    'expression': 'SUM(quantity_sold)'
                }
            ]
        }
    ]
    
    return datasets
```

### Sample RLS Configuration

```python
# Sample RLS rules for organization
RLS_RULES = [
    {
        'name': 'Regional Manager - West',
        'tables': ['data_transaksi_sparepart', 'vw_product_performance'],
        'clause': "cabang IN ('JAKARTA', 'BANDUNG', 'BOGOR')",
        'roles': ['Regional Manager West']
    },
    {
        'name': 'Regional Manager - East', 
        'tables': ['data_transaksi_sparepart', 'vw_product_performance'],
        'clause': "cabang IN ('SURABAYA', 'MALANG', 'BALI')",
        'roles': ['Regional Manager East']
    },
    {
        'name': 'National Director',
        'tables': ['data_transaksi_sparepart', 'vw_product_performance'],
        'clause': "1=1",  # Access to all data
        'roles': ['National Director']
    }
]
```

## Troubleshooting Data Access Issues

### Common Errors and Solutions

```python
# Error: Permission denied for table
"""
Solution: 
1. Ensure database user has SELECT permission
2. Check RLS rules not conflicting
3. Verify schema permissions
"""

# Error: Cannot explore dataset
"""
Solution:
1. Check user has 'can explore' permission  
2. Verify dataset published
3. Check database connection valid
"""

# Error: Query timeout
"""
Solution:
1. Increase SQLLAB_TIMEOUT
2. Optimize database query
3. Add appropriate indexes
"""
```

### Debugging RLS Issues

```sql
-- Test RLS clause directly
SELECT * FROM mb.data_transaksi_sparepart 
WHERE cabang = 'JAKARTA'  -- RLS clause

-- Check user context
SELECT current_user, session_user;

-- Verify table permissions
SELECT grantee, privilege_type 
FROM information_schema.role_table_grants 
WHERE table_name = 'data_transaksi_sparepart';
```

### Monitoring Data Access

```python
# Log data access attempts
import logging
from flask import request, g

def log_data_access(dataset, user, query):
    logging.info(f"Data access - User: {user}, Dataset: {dataset}, Query: {query}")

# Custom query logger
class QueryLogger:
    def before_cursor_execute(self, conn, cursor, statement, parameters, context, executemany):
        if hasattr(g, 'user') and g.user is not None:
            logging.info(f"Query by {g.user.username}: {statement}")
```

## Best Practices for Data Access

### Security Best Practices
- Always use read-only database user
- Implement RLS for multi-tenant data
- Regular audit user permissions
- Monitor suspicious query patterns

### Performance Best Practices
- Use materialized views for complex queries
- Implement appropriate database indexes
- Configure connection pooling
- Setup query timeouts

### Management Best Practices
- Document all RLS rules
- Version control dataset configurations
- Regular cleanup unused datasets
- Monitor dataset usage metrics

With these configurations, you can granularly control data access in Apache Superset and ensure data security according to organizational business requirements.
