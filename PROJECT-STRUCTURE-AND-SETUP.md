# Project Structure and Setup Guide - romanchaa997

## Table of Contents
1. [Overview](#overview)
2. [Directory Structure](#directory-structure)
3. [Project Descriptions](#project-descriptions)
4. [Setup Instructions](#setup-instructions)
5. [Technology Stack](#technology-stack)
6. [Development Workflow](#development-workflow)
7. [Troubleshooting](#troubleshooting)

## Overview

This document provides a comprehensive guide to the structure, setup, and organization of all projects in the romanchaa997 portfolio. Each project follows modern Python development practices with clear separation of concerns, containerization, and automated testing.

## Directory Structure

```
romanchaa997/
├── README.md                              # Profile overview
├── MASTER-TECH-STACK-INDEX.md            # Central tech stack reference
├── PROJECT-STRUCTURE-AND-SETUP.md        # This file
├── TECH-STACK-IMPROVEMENTS.md            # Strategic improvements
├── TECH-STACK-QUICK-START.md             # Implementation guide
├── REPOSITORY-AUDIT-TEMPLATE.csv         # Audit framework
├── PYTHON-PYDANTIC-V2-MIGRATION.md       # Migration guide
├── ARCHITECTURE.md                       # System architecture
├── CONTRIBUTING.md                       # Contribution guidelines
├── SECURITY.md                           # Security policies
├── ECOSYSTEM_SETUP.md                    # Ecosystem setup
├── INVESTOR.md                           # Investor information
├── NEXT_STEPS.ua.md                      # Roadmap
├── CASE_STUDIES.md                       # Case studies
└── .github/
    ├── workflows/
    │   ├── tests.yml                     # Automated testing
    │   ├── linting.yml                   # Code quality checks
    │   └── deploy.yml                    # Deployment pipeline
    └── CODEOWNERS                        # Code ownership rules
```

## Project Descriptions

### 1. Bakhmach Business Hub
**Repository**: `romanchaa997/Bakhmach-Business-Hub`
**Status**: Production Ready
**Technologies**: Python, Django, PostgreSQL, Redis, Docker

**Purpose**: Comprehensive business intelligence and management platform for Bakhmach region

**Key Features**:
- Business data aggregation and analysis
- Real-time dashboard with metrics
- Multi-tenant support with role-based access
- Advanced caching layer
- API-first architecture

**Structure**:
```
Bakhmach-Business-Hub/
├── app/
│   ├── models/          # Django ORM models
│   ├── views/           # API endpoints
│   ├── serializers/      # DRF serializers
│   ├── migrations/       # Database migrations
│   └── tests/            # Test suite
├── config/              # Django settings
├── static/              # Static files
├── docker-compose.yml   # Container orchestration
├── requirements.txt     # Python dependencies
├── Dockerfile          # Container definition
└── manage.py           # Django CLI
```

**Setup**:
```bash
# Clone repository
git clone https://github.com/romanchaa997/Bakhmach-Business-Hub.git
cd Bakhmach-Business-Hub

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Create .env file
cp .env.example .env

# Run migrations
python manage.py migrate

# Create superuser
python manage.py createsuperuser

# Run development server
python manage.py runserver
```

### 2. Smart Bakhmach IoT Infrastructure
**Repository**: `romanchaa997/smart-bakhmach-iot-infrastructure`
**Status**: Production Ready
**Technologies**: Python, FastAPI, MQTT, InfluxDB, Grafana, Docker

**Purpose**: IoT data collection, processing, and visualization for smart city initiatives

**Key Features**:
- Real-time sensor data collection
- MQTT protocol support
- Time-series data storage
- Advanced visualization dashboards
- Alert and notification system
- Historical data analysis

**Structure**:
```
smart-bakhmach-iot-infrastructure/
├── src/
│   ├── api/             # FastAPI routes
│   ├── models/          # Pydantic models
│   ├── processors/      # Data processors
│   ├── services/        # Business logic
│   └── utils/           # Utilities
├── mqtt/                # MQTT client config
├── dashboards/          # Grafana dashboards
├── tests/               # Test suite
├── docker-compose.yml   # Service orchestration
├── requirements.txt     # Dependencies
└── Dockerfile          # Container definition
```

**Setup**:
```bash
# Clone repository
git clone https://github.com/romanchaa997/smart-bakhmach-iot-infrastructure.git
cd smart-bakhmach-iot-infrastructure

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Create environment file
cp .env.example .env

# Start services with Docker Compose
docker-compose up -d

# Run migrations if needed
python -m alembic upgrade head

# Start FastAPI server
uvicorn src.main:app --reload
```

## Setup Instructions

### Prerequisites
- Python 3.10 or higher
- Git
- Docker and Docker Compose (optional but recommended)
- PostgreSQL 14+ (for database projects)
- Redis 7+ (for caching)

### Common Setup Steps

#### 1. Clone Repository
```bash
git clone https://github.com/romanchaa997/PROJECT_NAME.git
cd PROJECT_NAME
```

#### 2. Create Virtual Environment
```bash
# Using venv
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate  # Windows
```

#### 3. Install Dependencies
```bash
# Basic installation
pip install -r requirements.txt

# With development dependencies
pip install -r requirements-dev.txt

# For specific project groups
pip install -r requirements-test.txt   # Testing
pip install -r requirements-lint.txt   # Linting
```

#### 4. Environment Configuration
```bash
# Copy example environment file
cp .env.example .env

# Edit .env with your values
nano .env
```

#### 5. Database Setup
```bash
# Run migrations
python manage.py migrate              # Django
python -m alembic upgrade head        # Alembic

# Create initial data
python manage.py loaddata fixtures/
```

#### 6. Run Tests
```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_models.py

# Run with coverage
pytest --cov=src
```

### Docker Setup

#### Using Docker Compose
```bash
# Build and start services
docker-compose up -d

# View logs
docker-compose logs -f

# Run commands in container
docker-compose exec web python manage.py migrate

# Stop services
docker-compose down
```

#### Building Docker Image
```bash
# Build image
docker build -t romanchaa997/PROJECT_NAME:latest .

# Run container
docker run -p 8000:8000 romanchaa997/PROJECT_NAME:latest

# With environment variables
docker run -e DATABASE_URL=postgresql://... romanchaa997/PROJECT_NAME:latest
```

## Technology Stack

### Backend
- **Python 3.10+** - Primary programming language
- **Django** - Web framework for traditional apps
- **FastAPI** - Modern async API framework
- **Pydantic V2** - Data validation and settings management
- **SQLAlchemy** - ORM for database operations
- **Alembic** - Database migration tool

### Databases
- **PostgreSQL 14+** - Primary relational database
- **Redis 7+** - In-memory caching
- **InfluxDB** - Time-series data storage
- **MongoDB** (optional) - Document database

### DevOps & Infrastructure
- **Docker** - Containerization
- **Docker Compose** - Multi-container orchestration
- **Kubernetes** - Container orchestration (optional)
- **GitHub Actions** - CI/CD automation
- **Prometheus** - Metrics collection
- **Grafana** - Visualization and dashboards

### Testing & Quality
- **pytest** - Testing framework
- **pytest-cov** - Code coverage
- **unittest** - Standard library testing
- **Black** - Code formatter
- **Flake8** - Linting
- **MyPy** - Static type checking
- **isort** - Import sorting

### Documentation
- **Sphinx** - Documentation generation
- **MkDocs** - Documentation site
- **OpenAPI/Swagger** - API documentation

## Development Workflow

### Feature Development
1. Create feature branch: `git checkout -b feature/feature-name`
2. Make changes following code style guidelines
3. Write tests for new functionality
4. Run linters and formatters: `make lint` or `black . && flake8`
5. Ensure all tests pass: `pytest`
6. Commit with descriptive messages
7. Push branch and create pull request
8. Address review comments
9. Merge to main branch after approval

### Code Style
- Follow PEP 8 guidelines
- Use Black for code formatting (line length: 88)
- Use type hints throughout
- Maximum line length: 88 characters
- Docstring format: Google style

### Commit Messages
```
format: <type>(<scope>): <subject>

<body>

<footer>

Types: feat, fix, docs, style, refactor, perf, test, chore
```

### Pull Request Process
1. Update documentation
2. Add/update tests
3. Ensure CI/CD passes
4. Get at least 2 approvals
5. Merge with squash option

## Troubleshooting

### Common Issues

#### Port Already in Use
```bash
# Find process using port
lsof -i :8000

# Kill process
kill -9 <PID>

# Or use different port
python manage.py runserver 8001
```

#### Database Connection Issues
```bash
# Verify DATABASE_URL in .env
echo $DATABASE_URL

# Test connection
psql $DATABASE_URL

# Check PostgreSQL service
sudo systemctl status postgresql
```

#### Python Version Mismatch
```bash
# Check current version
python --version

# Use pyenv for version management
pyenv install 3.10.5
pyenv local 3.10.5
```

#### Dependency Conflicts
```bash
# Upgrade pip
pip install --upgrade pip

# Clear cache and reinstall
pip cache purge
pip install -r requirements.txt --force-reinstall
```

### Getting Help
- Check [CONTRIBUTING.md](./CONTRIBUTING.md) for contribution guidelines
- Review [ARCHITECTURE.md](./ARCHITECTURE.md) for system design
- See [TECH-STACK-QUICK-START.md](./TECH-STACK-QUICK-START.md) for implementation details
- Consult [MASTER-TECH-STACK-INDEX.md](./MASTER-TECH-STACK-INDEX.md) for comprehensive documentation

## Related Documentation
- [Master Tech Stack Index](./MASTER-TECH-STACK-INDEX.md)
- [Technology Improvements](./TECH-STACK-IMPROVEMENTS.md)
- [Quick Start Guide](./TECH-STACK-QUICK-START.md)
- [Architecture Documentation](./ARCHITECTURE.md)
- [Contributing Guidelines](./CONTRIBUTING.md)

---

**Last Updated**: January 2026
**Maintained By**: romanchaa997
**Status**: Active Development
