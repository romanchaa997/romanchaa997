# Database Optimization Guide

## Executive Summary
Comprehensive PostgreSQL and NoSQL database optimization strategy for production systems, targeting 50-70% improvement in query performance and throughput.

## Part 1: PostgreSQL Configuration

### 1.1 Server Configuration Parameters

```postgresql
-- Memory Configuration (adjust based on system RAM)
shared_buffers = 4GB              -- 25% of system RAM
effective_cache_size = 12GB       -- 75% of system RAM
maintenance_work_mem = 1GB        -- For VACUUM, CREATE INDEX
work_mem = 50MB                   -- Per-operation memory

-- Connection Configuration
max_connections = 200
res_send_buffer_size = 8MB
wal_buffers = 16MB

-- Checkpoint Configuration
wal_level = replica
max_wal_size = 4GB
wal_keep_size = 2GB
checkpoint_completion_target = 0.9

-- Query Execution
random_page_cost = 1.1            -- For SSD storage
effective_io_concurrency = 200    -- For SSD storage
```

### 1.2 Connection Pooling with PgBouncer

```ini
[databases]
app_prod = host=localhost port=5432 dbname=app_prod

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 10
reserve_pool_size = 5
reserve_pool_timeout = 3
server_lifetime = 3600
server_idle_timeout = 600
server_connect_timeout = 15

[monitoring]
stats_period = 60
```

## Part 2: Indexing Strategy

### 2.1 Primary Key Strategy

- Use BIGSERIAL for primary keys (not UUIDs unless necessary)
- Consider BRIN indexes for time-series data
- Implement partitioning for large tables (>100GB)

### 2.2 Index Types

#### B-Tree Indexes (Default)
- For equality and range queries
- Most common and efficient

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_status_created ON users(status, created_at);
```

#### Hash Indexes
- For exact equality matches
- Faster than B-Tree for equality

```sql
CREATE INDEX idx_users_id_hash ON users USING HASH (id);
```

#### BRIN Indexes
- For time-series and large tables
- Minimal storage overhead

```sql
CREATE INDEX idx_events_timestamp ON events USING BRIN (timestamp);
```

#### GiST Indexes
- For geometric data and full-text search

```sql
CREATE INDEX idx_locations_geo ON locations USING GIST (coordinates);
```

### 2.3 Composite Indexes

- Order columns from most selective to least
- Match query WHERE and ORDER BY clauses

```sql
CREATE INDEX idx_orders_user_status_date ON orders(
    user_id,
    status,
    created_at
);

-- Covering indexes (include non-index columns)
CREATE INDEX idx_orders_covering ON orders(
    user_id,
    status
) INCLUDE (total_amount, created_at);
```

## Part 3: Query Optimization

### 3.1 Using EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
SELECT u.id, u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.email;
```

### 3.2 Common Query Issues & Fixes

#### N+1 Queries
- Problem: Multiple queries in a loop
- Solution: Use JOINs or batch loading

```sql
-- Bad: N+1 queries
SELECT * FROM users;
-- Then for each user: SELECT * FROM orders WHERE user_id = ?;

-- Good: Single query with JOIN
SELECT u.*, o.* 
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

#### Missing Indexes
- Use pg_stat_statements to identify slow queries
- Check Seq Scans in EXPLAIN output

```sql
-- Enable tracking
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slowest queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

#### Inefficient JOINs
- Ensure join columns are indexed
- Use INNER JOIN when possible (more optimization opportunities)
- Avoid JOINing on functions

### 3.3 Query Tuning Techniques

```sql
-- Use WHERE instead of HAVING when possible
SELECT user_id, COUNT(*) 
FROM orders
WHERE status = 'completed'
GROUP BY user_id
HAVING COUNT(*) > 5;

-- Limit joins to necessary columns
SELECT u.id, u.email, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email;

-- Use EXISTS for existence checks
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

## Part 4: Table Partitioning

### 4.1 Range Partitioning by Date

```sql
CREATE TABLE events (
    id BIGSERIAL,
    user_id BIGINT,
    event_type VARCHAR(50),
    created_at TIMESTAMP,
    data JSONB
) PARTITION BY RANGE (DATE_TRUNC('month', created_at));

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE INDEX idx_events_2024_01_user_id ON events_2024_01(user_id);
```

### 4.2 List Partitioning by Status

```sql
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT,
    status VARCHAR(20),
    amount DECIMAL(10,2)
) PARTITION BY LIST (status);

CREATE TABLE orders_active PARTITION OF orders
    FOR VALUES IN ('pending', 'processing');

CREATE TABLE orders_completed PARTITION OF orders
    FOR VALUES IN ('completed', 'shipped');
```

## Part 5: Materialized Views

```sql
-- Create materialized view for expensive aggregations
CREATE MATERIALIZED VIEW user_stats AS
SELECT 
    u.id,
    u.email,
    COUNT(o.id) as total_orders,
    SUM(o.amount) as total_spent,
    MAX(o.created_at) as last_order_date,
    AVG(o.amount) as avg_order_value
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email;

-- Create index on materialized view
CREATE INDEX idx_user_stats_id ON user_stats(id);
CREATE INDEX idx_user_stats_spent ON user_stats(total_spent DESC);

-- Refresh strategy (periodic or incremental)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;
```

## Part 6: Cache Strategy

### 6.1 Redis Caching

```python
import redis
import json
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_result(expire_time=3600):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = f"{func.__name__}:{args}:{kwargs}"
            
            # Try to get from cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Store in cache
            redis_client.setex(
                cache_key,
                expire_time,
                json.dumps(result)
            )
            return result
        return wrapper
    return decorator

@cache_result(expire_time=1800)
def get_user_orders(user_id):
    # Query database
    pass
```

### 6.2 Application-Level Caching

- Cache frequently accessed data
- Cache expensive computations
- Cache database query results
- Use cache-aside pattern
- Implement cache invalidation on updates

## Part 7: Replication & High Availability

### 7.1 Streaming Replication Setup

```postgresql
-- On primary
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

-- On replica
standby_mode = 'on'
primary_conninfo = 'host=primary_ip port=5432 user=replicator password=***'
restore_command = 'cp /wal_archive/%f %p'
```

### 7.2 Synchronous Replication

```postgresql
-- On primary
synchronous_standby_names = 'standby1,standby2'
```

## Part 8: Maintenance Tasks

### 8.1 VACUUM Strategy

```postgresql
-- Regular maintenance
VACUUM ANALYZE;

-- Aggressive vacuum for bloated tables
VACUUM FULL ANALYZE;

-- AutoVACUUM configuration
Autovacuum = on
Autovacuum_max_workers = 4
Autovacuum_naptime = 30s
```

### 8.2 ANALYZE Statistics

```postgresql
-- Update table statistics
ANALYZE users;

-- Update all statistics
ANALYZE;

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

## Part 9: Monitoring & Troubleshooting

### 9.1 Key Metrics to Monitor

- Cache hit ratio: >90%
- Transaction rate: Track trends
- Lock contention: Minimize wait times
- Slow queries: <100ms p99
- Replication lag: <1 second

### 9.2 Performance Queries

```sql
-- Active connections
SELECT count(*) FROM pg_stat_activity;

-- Long-running queries
SELECT pid, query, query_start
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '5 minutes';

-- Unused indexes
SELECT schemaname, tablename, indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Table bloat
SELECT schemaname, tablename, round(bloat_ratio*100) as bloat_percent
FROM pgstattuple_approx;
```

## Success Metrics

- 50-70% improvement in query performance
- Cache hit ratio >90%
- Database CPU <70% under normal load
- Query p99 <100ms
- Zero query timeouts
- Replication lag <500ms
