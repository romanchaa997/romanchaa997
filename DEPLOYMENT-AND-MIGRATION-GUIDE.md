# Deployment & Migration Guide - romanchaa997 Tech Stack

## Overview

This document provides step-by-step procedures for deploying and migrating tech stack improvements across all projects in the romanchaa997 portfolio, with emphasis on zero-downtime deployment and data integrity.

## Pre-Deployment Checklist

### Code Quality
- [ ] All tests passing (100% success rate)
- [ ] Code coverage >= 85%
- [ ] All linting checks passing
- [ ] No critical vulnerabilities (CVSS >= 9.0)
- [ ] All type hints in place
- [ ] Documentation updated

### Infrastructure
- [ ] Staging environment mirrors production
- [ ] Database backups completed
- [ ] Rollback plan documented
- [ ] Monitoring and alerts configured
- [ ] Deployment scripts tested
- [ ] Team members on standby

### Communication
- [ ] Stakeholders notified
- [ ] Maintenance window scheduled
- [ ] Escalation contacts ready
- [ ] Status page prepared
- [ ] Incident response team ready

---

## Blue-Green Deployment Strategy

### Overview
Deploy new version to fresh infrastructure while old version remains active. Switch traffic after validation.

### Procedure

**Step 1: Prepare Green Environment** (30 mins)
```bash
# Clone current infrastructure
docker-compose -f docker-compose.green.yml up -d

# Run database migrations
docker-compose -f docker-compose.green.yml exec web alembic upgrade head

# Seed test data
docker-compose -f docker-compose.green.yml exec web python seed_data.py
```

**Step 2: Deploy New Code** (20 mins)
```bash
# Build new image
docker build -t myapp:v2.0.0 .

# Push to registry
docker push myapp:v2.0.0

# Deploy to green environment
docker-compose -f docker-compose.green.yml down
docker-compose -f docker-compose.green.yml up -d
```

**Step 3: Validation** (30 mins)
```bash
# Run smoke tests
pytest tests/integration/ --smoke

# Check API endpoints
curl http://localhost:8001/health

# Verify database
python check_database.py

# Load testing
locust -f locustfile.py --host=http://localhost:8001 -u 100 -r 10 --run-time 5m
```

**Step 4: Traffic Switch** (5 mins)
```bash
# Update load balancer
nginx_switch_upstream.sh green

# Verify traffic
watch 'curl http://api.example.com/health'
```

**Step 5: Monitor** (30 mins)
- Monitor error logs: `docker logs -f green_app`
- Check metrics: `prometheus_query('http_requests_total')`
- Monitor performance: `grafana_dashboard('performance')`
- Check database: `SELECT COUNT(*) FROM audit_log WHERE created_at > NOW() - INTERVAL 5 min`

**Step 6: Cleanup** (10 mins)
```bash
# Keep blue environment running for 1 hour
sleep 3600

# If all clear, stop blue
docker-compose -f docker-compose.blue.yml down

# Archive logs
tar czf logs/deployment-$(date +%Y%m%d-%H%M%S).tar.gz blue_app_logs/
```

### Rollback Procedure
```bash
# If issues detected, switch traffic back to blue
nginx_switch_upstream.sh blue

# Monitor blue environment
docker logs -f blue_app

# Once stable, document issues and create post-mortem
```

---

## Pydantic V2 Migration

### Phase 1: Preparation

**1.1 Audit Current Code** (4 hours)
```bash
# Find all Pydantic imports
grep -r "from pydantic import" --include="*.py"

# Find all BaseModel classes
grep -r "class.*BaseModel" --include="*.py"

# Count validators
grep -r "@validator" --include="*.py" | wc -l
```

**1.2 Create Migration Branch**
```bash
git checkout -b feature/pydantic-v2-migration
```

**1.3 Update requirements.txt**
```diff
- pydantic==1.10.x
+ pydantic==2.0.x
```

### Phase 2: Code Updates

**2.1 Update Model Definitions**

*Before (Pydantic V1)*
```python
from pydantic import BaseModel, validator

class User(BaseModel):
    name: str
    email: str
    age: int
    
    @validator('email')
    def validate_email(cls, v):
        if '@' not in v:
            raise ValueError('Invalid email')
        return v
    
    class Config:
        orm_mode = True
```

*After (Pydantic V2)*
```python
from pydantic import BaseModel, field_validator, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    
    name: str
    email: str
    age: int
    
    @field_validator('email')
    @classmethod
    def validate_email(cls, v):
        if '@' not in v:
            raise ValueError('Invalid email')
        return v
```

**2.2 Update Validators** (2-3 hours)
```python
# V1 Syntax
@root_validator(pre=True)
def validate_all(cls, values):
    ...

# V2 Syntax  
@model_validator(mode='before')
@classmethod
def validate_all(cls, values):
    ...
```

**2.3 Update Model Serialization** (2-3 hours)
```python
# V1 Syntax
model.dict()
model.json()

# V2 Syntax
model.model_dump()
model.model_dump_json()
```

### Phase 3: Testing

**3.1 Unit Tests** (2-3 hours)
```bash
pytest tests/models/ -v
```

**3.2 Integration Tests** (2-3 hours)
```bash
pytest tests/api/ -v
```

**3.3 Load Testing** (1 hour)
```bash
locust -f locustfile.py --host=http://localhost:8000 -u 50 -r 5 --run-time 10m
```

### Phase 4: Deployment

**4.1 Staging Deployment**
```bash
git push origin feature/pydantic-v2-migration
# Create PR, get 2+ approvals
# Merge to staging branch
# Deploy to staging
```

**4.2 Production Deployment**
```bash
# After 48 hours of staging validation
git merge staging main
# Deploy using blue-green strategy above
```

---

## FastAPI Migration

### For Django REST Projects

**Step 1: Parallel API Implementation**
```python
# FastAPI app alongside Django
from fastapi import FastAPI
from fastapi.middleware.wsgi import WSGIMiddleware
from django.core.wsgi import get_wsgi_application

app = FastAPI()
django_app = WSGIMiddleware(get_wsgi_application())

# Route FastAPI endpoints
@app.get("/api/v2/users")
async def get_users():
    ...

# Route Django endpoints
app.mount("/api/v1", django_app)
```

**Step 2: Gradual Migration**
- Week 1: Implement 20% of endpoints in FastAPI
- Week 2: Implement 40% of endpoints
- Week 3: Implement 60% of endpoints
- Week 4: Implement 80% of endpoints
- Week 5: Implement remaining 20%, remove Django REST

**Step 3: Load Balancing**
```nginx
upstream api_backend {
    # Route by endpoint pattern
    server fastapi:8000;
    server django:8001;
}

server {
    location /api/v2 {
        proxy_pass http://fastapi:8000;
    }
    
    location /api/v1 {
        proxy_pass http://django:8001;
    }
}
```

---

## Database Migration

### Pre-Migration
```bash
# Backup database
pg_dump -U postgres dbname > backup_$(date +%Y%m%d_%H%M%S).sql

# Verify backup
file backup_*.sql
du -h backup_*.sql
```

### Migration Procedure
```bash
# 1. Create new database
createdb -U postgres dbname_v2

# 2. Run migrations in new database
alembic upgrade head -d postgresql://user:password@localhost/dbname_v2

# 3. Seed test data
python seed_data.py --db postgresql://user:password@localhost/dbname_v2

# 4. Run validation queries
python validate_migration.py

# 5. Perform cutover
# Update connection string in production
# Monitor for errors

# 6. Cleanup
# Keep old database for 30 days
# Then: dropdb -U postgres dbname
```

---

## Monitoring & Validation

### Key Metrics
- API response time (target: < 200ms p95)
- Error rate (target: < 0.1%)
- Database query time (target: < 100ms p95)
- CPU usage (target: < 80%)
- Memory usage (target: < 85%)
- Disk I/O (target: < 70%)

### Validation Queries
```sql
-- Check data integrity
SELECT COUNT(*) FROM users WHERE created_at > NOW() - INTERVAL '1 day';
SELECT COUNT(*) FROM transactions WHERE status != 'completed';

-- Check for errors
SELECT level, COUNT(*) FROM logs WHERE created_at > NOW() - INTERVAL '1 hour' GROUP BY level;

-- Check performance
SELECT AVG(execution_time), MAX(execution_time) FROM queries WHERE created_at > NOW() - INTERVAL '1 hour';
```

---

## Incident Response

### If Issues Detected

**Level 1: Minor Issues (Non-Critical)**
- Continue deployment
- Create issue ticket
- Add to post-mortem

**Level 2: Moderate Issues (Degraded Performance)**
- Initiate rollback if > 5% error rate
- Page on-call engineer
- Start incident response

**Level 3: Critical Issues (Complete Outage)**
- Immediate rollback
- All hands on deck
- Direct customer communication
- Post-incident review within 24 hours

---

## Post-Deployment

**Validation** (2 hours)
- [ ] All endpoints responding
- [ ] No spike in error logs
- [ ] Performance metrics normal
- [ ] Database responding normally
- [ ] Users reporting normal functionality

**Documentation** (1 hour)
- [ ] Update changelog
- [ ] Document any issues
- [ ] Update runbooks
- [ ] Update disaster recovery plan

**Team Debrief** (30 mins)
- [ ] What went well
- [ ] What could improve
- [ ] Action items for next deployment
- [ ] Update playbooks

---

**Last Updated**: January 3, 2026, 4 AM EET
**Owner**: DevOps Engineer
**Status**: Active
**Next Review**: Before each deployment
