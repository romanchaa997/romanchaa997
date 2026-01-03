# ⚡ Tech Stack Quick Start Implementation Guide

## 🚀 Week 1 Checklist (Days 1-7)

### Day 1-2: Audit & Planning
- [ ] Review all 83 repos in your portfolio
- [ ] Identify repos by language: Python, TypeScript, Mixed
- [ ] Create a spreadsheet with current Python/Node versions
- [ ] Document current FastAPI, React, and other key versions

### Day 3-4: Setup Shared Infrastructure
```bash
# 1. Create .github/workflows/standard-lint.yml
- black (Python formatting)
- ruff (Python linting)
- mypy (Type checking)
- eslint (TypeScript linting)
- prettier (Code formatting)

# 2. Create pyproject.toml template for all Python projects
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "project-name"
version = "0.1.0"
requires-python = ">=3.11"

# 3. Create pre-commit config
repos:
  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black
  - repo: https://github.com/charliermarsh/ruff
    rev: 0.1.8
    hooks:
      - id: ruff
        args: [--fix]
```

### Day 5-7: Start with Bakhmach-Business-Hub
- [ ] Clone the repo locally
- [ ] Upgrade Python to 3.11 (check pyenv)
- [ ] Create requirements.txt with pinned versions
- [ ] Run `pip install -r requirements.txt`
- [ ] Setup pre-commit hooks
- [ ] Add .pre-commit-config.yaml

---

## 🎯 Week 2-3: Core Backend Updates

### Update FastAPI & Dependencies
```bash
# Create/update requirements.txt
fastapi==0.109.0
uvicorn==0.25.0
pydantic==2.5.0
pydantic-settings==2.1.0
python-dotenv==1.0.0

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0
httpx==0.25.2  # For testing FastAPI

# Code quality
black==23.12.0
ruff==0.1.8
mypy==1.7.1
bandit==1.7.5  # Security
pylint==3.0.3

# ORM & Database
sqlalchemy==2.0.23
alembic==1.12.1
psycopg[binary]==3.18.0  # PostgreSQL
motor==3.3.2  # Async MongoDB (if using)

# Caching & Sessions
redis==5.0.1
python-redis-lock==4.0.1

# Auth
pyjwt==2.8.1
bcrypt==4.1.1
python-multipart==0.0.6
```

### Update FastAPI app.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

# Lifespan context
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("Application startup")
    yield
    # Shutdown
    print("Application shutdown")

app = FastAPI(
    title="Your API",
    version="1.0.0",
    lifespan=lifespan,
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure properly
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "app:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
    )
```

### Test the Update
```bash
pip install -r requirements.txt
pytest tests/ --cov=src --cov-report=term-missing
black src/ tests/
ruff check src/ tests/
mypy src/
```

---

## 🎨 Week 4: Frontend Quick Setup

### For Next.js 14 migration
```bash
# If starting new:
npx create-next-app@latest my-app --typescript

# Or upgrade existing:
npm install next@latest react@latest react-dom@latest

# Add recommended packages:
npm install \
  @hookform/resolvers \
  axios \
  clsx \
  date-fns \
  next-auth \
  zustand \
  zod \
  tailwindcss \
  @tailwindcss/typography \
  postcss \
  autoprefixer
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "strict": true,
    "jsx": "preserve",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## 🔧 Essential Git Hooks Setup

### .pre-commit-config.yaml (Python repos)
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black
        args: [--line-length=100]

  - repo: https://github.com/charliermarsh/ruff
    rev: 0.1.8
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.7.1
    hooks:
      - id: mypy
        args: [--strict]
        additional_dependencies: [types-all]
```

### Install in your repo
```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files  # Test all files
```

---

## 📊 GitHub Actions CI/CD Template

### .github/workflows/test.yml
```yaml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.11', '3.12']

    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Lint with black
        run: black --check src/ tests/
      
      - name: Lint with ruff
        run: ruff check src/ tests/
      
      - name: Type check with mypy
        run: mypy src/
      
      - name: Run tests
        run: pytest tests/ --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
```

---

## 🐳 Docker Multi-Stage Build Template

### Dockerfile (Python FastAPI)
```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app
COPY requirements.txt .

RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

# Copy Python dependencies from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY src/ ./src/
COPY .env.example .env

# Add to PATH
ENV PATH=/root/.local/bin:$PATH

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/app
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app
    volumes:
      - postgres_data:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## ✅ Immediate Actions (This Week)

### 1. Standardize pyproject.toml
```bash
# Copy to ALL Python repos
cp template-pyproject.toml Bakhmach-Business-Hub/pyproject.toml
cp template-pyproject.toml smart-bakhmach-iot/pyproject.toml
# ... etc
```

### 2. Update Python Version
```bash
# Check current:
python --version

# Install 3.11 with pyenv:
pyenv install 3.11.7
pyenv local 3.11.7
python --version  # Should be 3.11.7
```

### 3. Create .pre-commit-config.yaml in each repo
```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files
```

### 4. Run first tests
```bash
pip install -r requirements.txt
pytest tests/ --cov=src
```

### 5. Commit & Push
```bash
git add -A
git commit -m "chore: Update tech stack - Python 3.11, pre-commit hooks, pinned dependencies"
git push origin main
```

---

## 📚 Quick Reference Links

### Python
- [FastAPI 0.109 Docs](https://fastapi.tiangolo.com/)
- [Pydantic V2 Migration](https://docs.pydantic.dev/latest/api/pydantic_v1/)
- [SQLAlchemy 2.0 Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [pytest Documentation](https://docs.pytest.org/)

### TypeScript/Next.js
- [Next.js 14 Docs](https://nextjs.org/docs)
- [TypeScript 5.3 Handbook](https://www.typescriptlang.org/docs/)
- [TailwindCSS 3.4](https://tailwindcss.com/docs)
- [Vitest Docs](https://vitest.dev/)

### DevOps
- [GitHub Actions](https://docs.github.com/en/actions)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Pre-commit Hooks](https://pre-commit.com/)

---

## 🎯 Success Metrics

✅ **Week 1:** All 83 repos audited, shared templates created
✅ **Week 2:** Bakhmach-Business-Hub updated to Python 3.11
✅ **Week 3:** All Python deps pinned, CI/CD passing
✅ **Week 4:** Frontend repos have Next.js 14 + TypeScript strict mode
✅ **Ongoing:** 80%+ test coverage, green CI/CD builds

---

## 🚨 Troubleshooting

### Issue: Pre-commit hook failures
```bash
# Fix all files automatically
pre-commit run --all-files

# Bypass hooks (NOT recommended for production)
git commit --no-verify
```

### Issue: FastAPI import errors
```bash
pip install --upgrade fastapi
pip install --upgrade pydantic
python -c "import fastapi; print(fastapi.__version__)"
```

### Issue: Python 3.11 not found
```bash
# Using pyenv
pyenv install 3.11.7
pyenv local 3.11.7

# Using conda
conda create -n python311 python=3.11
conda activate python311
```

---

## 📞 Need Help?

Refer to the main `TECH-STACK-IMPROVEMENTS.md` for detailed explanations of each priority area.
