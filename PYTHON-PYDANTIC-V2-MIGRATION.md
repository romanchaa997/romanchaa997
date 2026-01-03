# 🔄 Python 3.11 + Pydantic V2 + FastAPI 0.109 Migration Guide

## Overview
This guide covers migrating Python projects from 3.9-3.10 + Pydantic V1 to Python 3.11 + Pydantic V2 + FastAPI 0.109+, focusing on your core projects.

---

## Phase 1: Preparation (Day 1-2)

### 1.1 Environment Setup
```bash
# Install Python 3.11
pyenv install 3.11.7
pyenv local 3.11.7
python --version  # Should output 3.11.7

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Upgrade pip & tools
pip install --upgrade pip setuptools wheel
pip install pip-tools  # For requirements management
```

### 1.2 Backup Current State
```bash
# Save current requirements
pip freeze > requirements.old.txt

# Create new branch for migration
git checkout -b feat/python3.11-pydantic-v2
```

---

## Phase 2: Dependency Upgrade (Day 3-4)

### 2.1 Update pyproject.toml
```toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "your-project"
version = "1.0.0"
requires-python = ">=3.11"

dependencies = [
    "fastapi==0.109.0",
    "uvicorn[standard]==0.25.0",
    "pydantic==2.5.0",
    "pydantic-settings==2.1.0",
    "python-dotenv==1.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest==7.4.3",
    "pytest-asyncio==0.21.1",
    "pytest-cov==4.1.0",
    "black==23.12.0",
    "ruff==0.1.8",
    "mypy==1.7.1",
]
db = [
    "sqlalchemy==2.0.23",
    "alembic==1.12.1",
    "psycopg[binary]==3.18.0",
]
```

### 2.2 Install New Dependencies
```bash
# Install all dependencies
pip install -e ".[dev,db]"

# Verify versions
python -c "import fastapi; import pydantic; print(f'FastAPI: {fastapi.__version__}, Pydantic: {pydantic.__version__}')"
```

---

## Phase 3: Code Migration (Day 5-7)

### 3.1 Pydantic V1 → V2 Changes

**Before (Pydantic V1):**
```python
from pydantic import BaseModel, Field, validator

class UserCreate(BaseModel):
    name: str
    email: str
    age: int = Field(..., ge=0, le=150)
    
    @validator('email')
    def email_must_contain_at(cls, v):
        if '@' not in v:
            raise ValueError('must contain @')
        return v
    
    class Config:
        orm_mode = True
```

**After (Pydantic V2):**
```python
from pydantic import BaseModel, Field, field_validator, ConfigDict

class UserCreate(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # replaces orm_mode
    
    name: str
    email: str
    age: int = Field(..., ge=0, le=150)
    
    @field_validator('email')
    @classmethod
    def email_must_contain_at(cls, v):
        if '@' not in v:
            raise ValueError('must contain @')
        return v
```

### 3.2 FastAPI + Pydantic V2 Integration
```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel, EmailStr
from typing import Annotated

app = FastAPI(
    title="My API",
    version="1.0.0",
    description="FastAPI 0.109 with Pydantic V2",
)

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    
@app.post("/items/")
async def create_item(item: Item) -> Item:
    """Create an item with automatic validation"""
    return item
```

### 3.3 Common Migration Issues

**Issue 1: orm_mode → from_attributes**
```python
# Old:
class Config:
    orm_mode = True

# New:
model_config = ConfigDict(from_attributes=True)
```

**Issue 2: validator → field_validator**
```python
# Old:
@validator('field')
def validate_field(cls, v):
    return v

# New:
@field_validator('field')
@classmethod
def validate_field(cls, v):
    return v
```

**Issue 3: Response models**
```python
# Old:
@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    pass

# New (same, but now uses Pydantic V2 serialization):
@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    pass
```

---

## Phase 4: Testing & Validation (Day 8-9)

### 4.1 Run Tests
```bash
# Update pytest for async support
pip install pytest-asyncio

# Run all tests
pytest tests/ -v --cov=src --cov-report=term-missing

# Expected: 80%+ coverage
```

### 4.2 Type Checking
```bash
# Run mypy for strict type checking
mypy src/ --strict

# Fix any type errors
```

### 4.3 Linting
```bash
# Format code
black src/ tests/

# Lint
ruff check src/ tests/ --fix

# Check imports
isort src/ tests/
```

---

## Phase 5: Database & ORM Updates (Day 10-12)

### 5.1 SQLAlchemy 2.0 + Async
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import declarative_base, sessionmaker
from sqlalchemy import Column, Integer, String

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/dbname"

engine = create_async_engine(DATABASE_URL, echo=False)
AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
```

### 5.2 Async Context in FastAPI
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await init_db()
    print("App started")
    yield
    # Shutdown
    await close_db()
    print("App closed")

app = FastAPI(lifespan=lifespan)
```

---

## Checklist

- [ ] Install Python 3.11
- [ ] Create pyproject.toml with new versions
- [ ] Update all imports (v1 → v2 Pydantic)
- [ ] Replace @validator with @field_validator
- [ ] Update orm_mode to from_attributes
- [ ] Run pytest (80%+ coverage required)
- [ ] Run mypy --strict
- [ ] Format with black & ruff
- [ ] Update SQLAlchemy to 2.0+ with async
- [ ] Test FastAPI endpoints
- [ ] Update CI/CD workflows
- [ ] Create PR and merge

---

## Rollback Plan

If issues occur:
```bash
# Revert to Python 3.10 + Pydantic V1
git checkout main
deactivate
rm -rf venv
pyenv local 3.10.13
python -m venv venv
source venv/bin/activate
pip install -r requirements.old.txt
```

---

## Performance Impact

- **Pydantic V2**: 2-5x faster validation
- **Python 3.11**: 10-20% faster execution
- **Combined impact**: 25-50% faster overall

---

## Resources

- [Pydantic V2 Migration Guide](https://docs.pydantic.dev/latest/)
- [FastAPI 0.109 Docs](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Python 3.11 Features](https://www.python.org/downloads/release/python-3110/)
