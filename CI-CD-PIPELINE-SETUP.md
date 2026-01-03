# CI/CD Pipeline Setup Guide

## Executive Summary
Comprehensive CI/CD pipeline implementation using GitHub Actions, Docker, and Kubernetes for continuous integration, testing, and deployment across all services.

## Part 1: GitHub Actions Workflow Structure

### 1.1 Workflow Organization

#### Main Pipeline Stages
1. **Trigger**: On push to branches, pull requests, or manual dispatch
2. **Build**: Compile code, run linters, build artifacts
3. **Test**: Unit tests, integration tests, end-to-end tests
4. **Security**: SAST, dependency scanning, container scanning
5. **Deploy**: Staging deployment with approval, production deployment
6. **Validation**: Smoke tests, health checks, monitoring verification

### 1.2 Workflow File Structure

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run linters
        run: |
          npm install
          npm run lint
          npm run format:check

  test:
    needs: lint
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install dependencies
        run: npm ci
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      - name: Run integration tests
        run: npm run test:integration
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to staging
        run: |
          kubectl apply -f k8s/staging/ --kubeconfig=${{ secrets.KUBECONFIG }}
          kubectl rollout status deployment/app -n staging --kubeconfig=${{ secrets.KUBECONFIG }}

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to production
        run: |
          kubectl apply -f k8s/production/ --kubeconfig=${{ secrets.KUBECONFIG }}
          kubectl rollout status deployment/app -n production --kubeconfig=${{ secrets.KUBECONFIG }}
      - name: Smoke tests
        run: npm run test:smoke
        env:
          API_URL: https://api.example.com
```

## Part 2: Testing Strategy

### 2.1 Test Pyramid

```
      E2E Tests (5-10%)
    Integration Tests (20-30%)
  Unit Tests (60-70%)
```

### 2.2 Test Execution

- **Unit Tests**: <5 minutes per service
- **Integration Tests**: 10-15 minutes total
- **E2E Tests**: 15-20 minutes total
- **Parallel Execution**: Run tests in parallel across services

### 2.3 Coverage Requirements

- Minimum coverage: 80%
- Critical paths: 95%
- Exclude: Third-party integrations, migrations
- Report: CodeCov integration with PR comments

## Part 3: Docker Build Optimization

### 3.1 Multi-Stage Dockerfile

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node healthcheck.js
CMD ["node", "server.js"]
```

### 3.2 Build Optimization

- Layer caching: Order dependencies from least to most frequently changed
- Minimal base image: Use alpine or distroless
- Image size: Target <200MB for base images
- Scan for vulnerabilities: Trivy/Snyk integration

## Part 4: Deployment Strategies

### 4.1 Blue-Green Deployment

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: app
    version: blue  # or green
  ports:
    - port: 80
      targetPort: 3000
```

### 4.2 Canary Deployment

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-canary
spec:
  hosts:
    - app.example.com
  http:
    - match:
        - uri:
            prefix: "/"
      route:
        - destination:
            host: app-v1
          weight: 90
        - destination:
            host: app-v2
          weight: 10
```

### 4.3 Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: app
          image: app:latest
```

## Part 5: Rollback Procedures

### 5.1 Automated Rollback Triggers

- Error rate exceeds 5%
- Response time p99 > 5s
- Pod crash loop backoff
- Memory/CPU exhaustion

### 5.2 Manual Rollback

```bash
# Kubernetes rollback
kubectl rollout history deployment/app -n production
kubectl rollout undo deployment/app -n production --to-revision=5
kubectl rollout status deployment/app -n production

# Verify rollback
kubectl get pods -n production -l app=app
kubectl logs -l app=app -n production --tail=50
```

## Part 6: Pipeline Security

### 6.1 Secret Management

- Use GitHub Secrets for sensitive data
- Rotate secrets every 90 days
- Never log secrets or API keys
- Use OIDC for credential exchange when possible

### 6.2 Access Control

- Require environment approval for production
- Use branch protection rules
- Code owners for sensitive files
- Audit all deployments with logs

## Part 7: Monitoring Pipeline Health

### 7.1 Pipeline Metrics

- Build time: Track trends
- Success rate: Target >95%
- Deployment frequency: Daily+
- Lead time: <24 hours from commit to production
- MTTR: <30 minutes

### 7.2 Pipeline Failures

- Notify on failures via Slack
- Create actionable runbooks
- Track common failure reasons
- Root cause analysis for repeated failures

## Part 8: Implementation Timeline

### Week 1: Foundation
- Set up GitHub Actions
- Create basic CI workflow
- Implement linting and testing

### Week 2: Build & Security
- Add Docker builds
- Implement security scanning
- Add code coverage reporting

### Week 3: Deployment
- Create staging deployment
- Add health checks
- Implement smoke tests

### Week 4: Production
- Production deployment pipeline
- Rollback procedures
- Monitoring and alerting

## Success Metrics

- Pipeline success rate: >95%
- Mean build time: <10 minutes
- Deployment frequency: Daily
- Lead time: <24 hours
- MTTR: <30 minutes
- Zero security vulnerabilities in production
