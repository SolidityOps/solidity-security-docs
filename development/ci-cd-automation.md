# CI/CD and Automation Guide

## Overview

This guide covers the complete CI/CD pipeline and automation strategies for the Solidity Security Platform. The platform uses GitHub Actions for continuous integration and deployment, with automated testing, quality gates, and deployment orchestration across all 11 services.

## CI/CD Architecture

### Multi-Service Pipeline Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                        CI/CD Pipeline                          │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Source    │  │   Build &   │  │       Deploy &          │  │
│  │   Control   │→ │    Test     │→ │       Monitor           │  │
│  │             │  │             │  │                         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                 │
│  • Git branches     • Multi-language   • Environment-specific │
│  • Pull requests    • Parallel jobs    • Health checks        │
│  • Code reviews     • Quality gates    • Rollback capability  │
└─────────────────────────────────────────────────────────────────┘
```

### Pipeline Components

1. **Source Control**: Git workflow with branch protection and code reviews
2. **Build & Test**: Multi-language builds with parallel execution
3. **Quality Gates**: Code quality, security, and coverage requirements
4. **Deployment**: Automated deployment to staging and production environments
5. **Monitoring**: Health checks, metrics, and alerting

## GitHub Actions Workflows

### Main CI/CD Pipeline

#### Primary Workflow (.github/workflows/ci-cd.yml)
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [published]

env:
  REGISTRY: ghcr.io
  REGISTRY_USERNAME: ${{ github.actor }}
  REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

jobs:
  # Detect changes to optimize builds
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      shared-library: ${{ steps.changes.outputs.shared-library }}
      python-services: ${{ steps.changes.outputs.python-services }}
      typescript-services: ${{ steps.changes.outputs.typescript-services }}
      rust-service: ${{ steps.changes.outputs.rust-service }}
      docs: ${{ steps.changes.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            shared-library:
              - 'solidity-security-shared/**'
            python-services:
              - 'solidity-security-api-service/**'
              - 'solidity-security-tool-integration/**'
              - 'solidity-security-intelligence-engine/**'
              - 'solidity-security-orchestration/**'
              - 'solidity-security-data-service/**'
              - 'solidity-security-notification/**'
            typescript-services:
              - 'solidity-security-ui-core/**'
              - 'solidity-security-dashboard/**'
              - 'solidity-security-findings/**'
              - 'solidity-security-analysis/**'
            rust-service:
              - 'solidity-security-contract-parser/**'
            docs:
              - 'docs/**'
              - '*.md'

  # Build shared library first (dependency for all services)
  build-shared-library:
    runs-on: ubuntu-latest
    needs: detect-changes
    if: needs.detect-changes.outputs.shared-library == 'true'
    steps:
      - uses: actions/checkout@v4

      - name: Set up Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          components: clippy, rustfmt

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
            node_modules
            ~/.cache/pip
          key: shared-library-${{ hashFiles('**/Cargo.lock', '**/package-lock.json', '**/requirements.txt') }}

      - name: Install dependencies
        working-directory: solidity-security-shared
        run: make install-deps

      - name: Build shared library
        working-directory: solidity-security-shared
        run: make build

      - name: Run tests
        working-directory: solidity-security-shared
        run: make test

      - name: Upload shared library artifacts
        uses: actions/upload-artifact@v3
        with:
          name: shared-library
          path: |
            solidity-security-shared/rust/target/release/
            solidity-security-shared/python/dist/
            solidity-security-shared/typescript/dist/
            solidity-security-shared/wasm-bindings/pkg/
          retention-days: 7

  # Build and test Python services in parallel
  build-python-services:
    runs-on: ubuntu-latest
    needs: [detect-changes, build-shared-library]
    if: always() && (needs.detect-changes.outputs.python-services == 'true' || needs.detect-changes.outputs.shared-library == 'true')
    strategy:
      matrix:
        service: [
          'api-service',
          'tool-integration',
          'intelligence-engine',
          'orchestration',
          'data-service',
          'notification'
        ]
        python-version: ['3.11']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Download shared library
        if: needs.build-shared-library.result == 'success'
        uses: actions/download-artifact@v3
        with:
          name: shared-library
          path: shared-library-artifacts

      - name: Cache Python dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: python-${{ matrix.service }}-${{ hashFiles(format('**/solidity-security-{0}/requirements/**', matrix.service)) }}

      - name: Install dependencies
        working-directory: solidity-security-${{ matrix.service }}
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements/test.txt

      - name: Lint and format check
        working-directory: solidity-security-${{ matrix.service }}
        run: |
          black --check src/ tests/
          isort --check src/ tests/
          ruff src/ tests/
          mypy src/

      - name: Run tests with coverage
        working-directory: solidity-security-${{ matrix.service }}
        run: |
          pytest --cov=src --cov-report=xml --cov-report=html --junitxml=pytest-results.xml

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./solidity-security-${{ matrix.service }}/coverage.xml
          flags: python-${{ matrix.service }}
          name: ${{ matrix.service }}

      - name: Build Docker image
        run: |
          docker build -t ${{ env.REGISTRY }}/solidity-security-${{ matrix.service }}:${{ github.sha }} \
            ./solidity-security-${{ matrix.service }}

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: python-${{ matrix.service }}-test-results
          path: |
            solidity-security-${{ matrix.service }}/pytest-results.xml
            solidity-security-${{ matrix.service }}/htmlcov/

  # Build and test TypeScript services in parallel
  build-typescript-services:
    runs-on: ubuntu-latest
    needs: [detect-changes, build-shared-library]
    if: always() && (needs.detect-changes.outputs.typescript-services == 'true' || needs.detect-changes.outputs.shared-library == 'true')
    strategy:
      matrix:
        service: ['ui-core', 'dashboard', 'findings', 'analysis']
        node-version: ['18']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: solidity-security-${{ matrix.service }}/package-lock.json

      - name: Download shared library
        if: needs.build-shared-library.result == 'success'
        uses: actions/download-artifact@v3
        with:
          name: shared-library
          path: shared-library-artifacts

      - name: Install dependencies
        working-directory: solidity-security-${{ matrix.service }}
        run: npm ci

      - name: Lint and type check
        working-directory: solidity-security-${{ matrix.service }}
        run: |
          npm run lint
          npm run type-check

      - name: Run tests
        working-directory: solidity-security-${{ matrix.service }}
        run: npm test -- --coverage --reporter=junit --outputFile=test-results.xml

      - name: Build application
        working-directory: solidity-security-${{ matrix.service }}
        run: npm run build

      - name: Run E2E tests
        working-directory: solidity-security-${{ matrix.service }}
        run: |
          npm run build
          npm run preview &
          sleep 5
          npm run test:e2e

      - name: Build Docker image
        if: matrix.service != 'ui-core'  # ui-core is a library, not an app
        run: |
          docker build -t ${{ env.REGISTRY }}/solidity-security-${{ matrix.service }}:${{ github.sha }} \
            ./solidity-security-${{ matrix.service }}

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: typescript-${{ matrix.service }}-test-results
          path: |
            solidity-security-${{ matrix.service }}/test-results.xml
            solidity-security-${{ matrix.service }}/coverage/

  # Build and test Rust service
  build-rust-service:
    runs-on: ubuntu-latest
    needs: [detect-changes, build-shared-library]
    if: always() && (needs.detect-changes.outputs.rust-service == 'true' || needs.detect-changes.outputs.shared-library == 'true')

    steps:
      - uses: actions/checkout@v4

      - name: Set up Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          components: clippy, rustfmt

      - name: Download shared library
        if: needs.build-shared-library.result == 'success'
        uses: actions/download-artifact@v3
        with:
          name: shared-library
          path: shared-library-artifacts

      - name: Cache Rust dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: rust-${{ hashFiles('**/Cargo.lock') }}

      - name: Check formatting
        working-directory: solidity-security-contract-parser
        run: cargo fmt --check

      - name: Lint with Clippy
        working-directory: solidity-security-contract-parser
        run: cargo clippy -- -D warnings

      - name: Run tests
        working-directory: solidity-security-contract-parser
        run: cargo test --all-features

      - name: Run benchmarks
        working-directory: solidity-security-contract-parser
        run: cargo bench

      - name: Build release
        working-directory: solidity-security-contract-parser
        run: cargo build --release

      - name: Build Docker image
        run: |
          docker build -t ${{ env.REGISTRY }}/solidity-security-contract-parser:${{ github.sha }} \
            ./solidity-security-contract-parser

  # Integration tests across services
  integration-tests:
    runs-on: ubuntu-latest
    needs: [build-python-services, build-typescript-services, build-rust-service]
    if: always() && !failure()

    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up test environment
        run: |
          cp .env.ci .env
          docker-compose -f docker-compose.test.yml pull
          docker-compose -f docker-compose.test.yml up -d --wait

      - name: Wait for services to be healthy
        run: |
          timeout 300 bash -c 'until curl -f http://localhost:8000/health; do sleep 5; done'
          timeout 300 bash -c 'until curl -f http://localhost:8002/health; do sleep 5; done'

      - name: Run integration tests
        run: |
          python scripts/integration_tests.py
          node scripts/e2e-integration.js

      - name: Run cross-language compatibility tests
        run: |
          python scripts/test_cross_language_compatibility.py

      - name: Upload integration test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: integration-test-results
          path: |
            integration-test-results.xml
            e2e-test-results/

  # Security scanning
  security-scan:
    runs-on: ubuntu-latest
    needs: detect-changes
    if: always()

    steps:
      - uses: actions/checkout@v4

      - name: Run Python security scan
        if: needs.detect-changes.outputs.python-services == 'true'
        run: |
          pip install safety bandit
          find . -name "requirements*.txt" -exec safety check -r {} \;
          find . -name "src" -type d -exec bandit -r {} \;

      - name: Run JavaScript security scan
        if: needs.detect-changes.outputs.typescript-services == 'true'
        run: |
          for service in ui-core dashboard findings analysis; do
            cd solidity-security-$service
            npm audit --audit-level=high
            cd ..
          done

      - name: Run Rust security scan
        if: needs.detect-changes.outputs.rust-service == 'true'
        run: |
          cargo install cargo-audit
          cd solidity-security-contract-parser
          cargo audit

      - name: Run container security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload security scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

  # Build and push Docker images
  build-and-push-images:
    runs-on: ubuntu-latest
    needs: [integration-tests, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    strategy:
      matrix:
        service: [
          'api-service',
          'tool-integration',
          'intelligence-engine',
          'orchestration',
          'data-service',
          'notification',
          'dashboard',
          'findings',
          'analysis',
          'contract-parser'
        ]

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ env.REGISTRY_USERNAME }}
          password: ${{ env.REGISTRY_PASSWORD }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/solidity-security-${{ matrix.service }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: ./solidity-security-${{ matrix.service }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  # Deploy to staging environment
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-push-images
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          # Update Kubernetes deployments
          kubectl set image deployment/api-service \
            api-service=${{ env.REGISTRY }}/solidity-security-api-service:${{ github.sha }} \
            --namespace=staging

          kubectl set image deployment/data-service \
            data-service=${{ env.REGISTRY }}/solidity-security-data-service:${{ github.sha }} \
            --namespace=staging

          # Wait for rollout to complete
          kubectl rollout status deployment/api-service --namespace=staging
          kubectl rollout status deployment/data-service --namespace=staging

      - name: Run smoke tests
        run: |
          curl -f https://staging-api.solidity-security.com/health
          curl -f https://staging-data.solidity-security.com/health

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Staging deployment completed successfully'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # Deploy to production (manual approval required)
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.event_name == 'release'
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        run: |
          # Blue-green deployment strategy
          scripts/deploy-production.sh ${{ github.sha }}

      - name: Run production smoke tests
        run: |
          scripts/production-smoke-tests.sh

      - name: Notify production deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment completed successfully'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Quality Gate Workflows

#### Code Quality Enforcement
```yaml
# .github/workflows/quality-gates.yml
name: Quality Gates

on:
  pull_request:
    branches: [main, develop]

jobs:
  enforce-quality:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for accurate analysis

      - name: Quality Gate - Code Coverage
        run: |
          # Ensure minimum coverage thresholds
          python scripts/check_coverage.py --minimum=80

      - name: Quality Gate - Test Results
        run: |
          # Ensure all tests pass
          if [ -f pytest-results.xml ]; then
            python scripts/check_test_results.py pytest-results.xml
          fi

      - name: Quality Gate - Performance
        run: |
          # Check for performance regressions
          python scripts/performance_regression_check.py

      - name: Quality Gate - Security
        run: |
          # Ensure no high/critical security vulnerabilities
          python scripts/security_check.py --severity=high

      - name: Quality Gate - Dependencies
        run: |
          # Check for outdated or vulnerable dependencies
          python scripts/dependency_check.py

      - name: Block merge if quality gates fail
        if: failure()
        run: |
          echo "Quality gates failed. Blocking merge."
          exit 1
```

#### Branch Protection Rules
```yaml
# Branch protection configuration (set via GitHub UI or API)
protection_rules:
  main:
    required_status_checks:
      strict: true
      contexts:
        - "build-shared-library"
        - "build-python-services"
        - "build-typescript-services"
        - "build-rust-service"
        - "integration-tests"
        - "security-scan"
        - "enforce-quality"
    enforce_admins: false
    required_pull_request_reviews:
      required_approving_review_count: 2
      dismiss_stale_reviews: true
      require_code_owner_reviews: true
    restrictions: null
```

## Release Automation

### Semantic Versioning and Release Management

#### Automated Release Workflow
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]
    paths-ignore:
      - 'docs/**'
      - '*.md'

jobs:
  release:
    runs-on: ubuntu-latest
    if: "!contains(github.event.head_commit.message, '[skip ci]')"

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.RELEASE_TOKEN }}
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install semantic-release
        run: |
          npm install -g semantic-release @semantic-release/changelog @semantic-release/git

      - name: Generate release
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_TOKEN }}
        run: |
          semantic-release

      - name: Update service versions
        if: steps.semantic-release.outputs.new-release-published == 'true'
        run: |
          NEW_VERSION="${{ steps.semantic-release.outputs.new-release-version }}"
          python scripts/update_service_versions.py $NEW_VERSION

      - name: Create release PR for version updates
        if: steps.semantic-release.outputs.new-release-published == 'true'
        run: |
          git checkout -b release/v${{ steps.semantic-release.outputs.new-release-version }}
          git add .
          git commit -m "chore: update service versions to v${{ steps.semantic-release.outputs.new-release-version }}"
          git push origin release/v${{ steps.semantic-release.outputs.new-release-version }}

          gh pr create \
            --title "chore: update service versions to v${{ steps.semantic-release.outputs.new-release-version }}" \
            --body "Automated version update following release" \
            --base main
```

#### Version Synchronization Script
```python
#!/usr/bin/env python3
# scripts/update_service_versions.py
import sys
import json
import toml
import re
from pathlib import Path

def update_python_version(service_path, version):
    """Update Python service version in pyproject.toml"""
    pyproject_path = service_path / "pyproject.toml"
    if pyproject_path.exists():
        data = toml.load(pyproject_path)
        data.setdefault("project", {})["version"] = version
        with open(pyproject_path, "w") as f:
            toml.dump(data, f)

def update_typescript_version(service_path, version):
    """Update TypeScript service version in package.json"""
    package_json_path = service_path / "package.json"
    if package_json_path.exists():
        with open(package_json_path, "r") as f:
            data = json.load(f)
        data["version"] = version
        with open(package_json_path, "w") as f:
            json.dump(data, f, indent=2)

def update_rust_version(service_path, version):
    """Update Rust service version in Cargo.toml"""
    cargo_toml_path = service_path / "Cargo.toml"
    if cargo_toml_path.exists():
        with open(cargo_toml_path, "r") as f:
            content = f.read()

        # Update version line
        content = re.sub(
            r'^version\s*=\s*"[^"]*"',
            f'version = "{version}"',
            content,
            flags=re.MULTILINE
        )

        with open(cargo_toml_path, "w") as f:
            f.write(content)

def main():
    if len(sys.argv) != 2:
        print("Usage: update_service_versions.py <version>")
        sys.exit(1)

    version = sys.argv[1]

    # Update shared library
    shared_lib_path = Path("solidity-security-shared")
    update_rust_version(shared_lib_path / "rust", version)
    update_python_version(shared_lib_path / "python", version)
    update_typescript_version(shared_lib_path / "typescript", version)

    # Update all services
    services = [
        # Python services
        ("solidity-security-api-service", update_python_version),
        ("solidity-security-tool-integration", update_python_version),
        ("solidity-security-intelligence-engine", update_python_version),
        ("solidity-security-orchestration", update_python_version),
        ("solidity-security-data-service", update_python_version),
        ("solidity-security-notification", update_python_version),

        # TypeScript services
        ("solidity-security-ui-core", update_typescript_version),
        ("solidity-security-dashboard", update_typescript_version),
        ("solidity-security-findings", update_typescript_version),
        ("solidity-security-analysis", update_typescript_version),

        # Rust service
        ("solidity-security-contract-parser", update_rust_version)
    ]

    for service_name, update_func in services:
        service_path = Path(service_name)
        if service_path.exists():
            update_func(service_path, version)
            print(f"Updated {service_name} to version {version}")

if __name__ == "__main__":
    main()
```

## Deployment Automation

### Environment Management

#### Environment Configuration
```yaml
# .env.staging
NODE_ENV=staging
DEBUG=false
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://user:pass@staging-db:5432/solidity_security
REDIS_URL=redis://staging-redis:6379/0

# External services
EXTERNAL_API_BASE_URL=https://staging-api.external.com
NOTIFICATION_WEBHOOK_URL=https://staging-webhooks.com/solidity-security

# Feature flags
ENABLE_ML_FEATURES=true
ENABLE_ADVANCED_ANALYSIS=false

# Monitoring
SENTRY_DSN=https://sentry.io/staging
PROMETHEUS_ENABLED=true
```

#### Kubernetes Deployment Templates
```yaml
# k8s/api-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.apiService.replicas }}
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api-service
        image: "{{ .Values.registry }}/solidity-security-api-service:{{ .Values.imageTag }}"
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: redis-config
              key: url
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: api-service
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
```

#### Deployment Scripts
```bash
#!/bin/bash
# scripts/deploy-production.sh
set -e

VERSION=$1
if [ -z "$VERSION" ]; then
  echo "Usage: deploy-production.sh <version>"
  exit 1
fi

echo "Deploying version $VERSION to production..."

# Update Helm values
helm upgrade --install solidity-security ./helm-chart \
  --namespace production \
  --set imageTag=$VERSION \
  --set environment=production \
  --values values.production.yaml \
  --wait --timeout=600s

# Verify deployment
echo "Verifying deployment..."
kubectl get pods -n production -l app.kubernetes.io/instance=solidity-security

# Run smoke tests
echo "Running smoke tests..."
./scripts/production-smoke-tests.sh

echo "Production deployment complete!"
```

### Health Checks and Monitoring

#### Service Health Endpoints
```python
# Python service health check
@app.get("/health")
async def health_check():
    """Comprehensive health check endpoint."""
    checks = {}
    overall_healthy = True

    # Database connectivity
    try:
        await database.execute("SELECT 1")
        checks["database"] = {"status": "healthy"}
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}
        overall_healthy = False

    # Redis connectivity
    try:
        await redis.ping()
        checks["redis"] = {"status": "healthy"}
    except Exception as e:
        checks["redis"] = {"status": "unhealthy", "error": str(e)}
        overall_healthy = False

    # Shared library availability
    try:
        from solidity_shared import RUST_AVAILABLE
        checks["shared_library"] = {
            "status": "healthy",
            "rust_acceleration": RUST_AVAILABLE
        }
    except Exception as e:
        checks["shared_library"] = {"status": "unhealthy", "error": str(e)}
        overall_healthy = False

    # External service dependencies
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(f"{EXTERNAL_API_BASE_URL}/health", timeout=5)
            if response.status_code == 200:
                checks["external_api"] = {"status": "healthy"}
            else:
                checks["external_api"] = {"status": "degraded", "status_code": response.status_code}
    except Exception as e:
        checks["external_api"] = {"status": "unhealthy", "error": str(e)}

    return {
        "status": "healthy" if overall_healthy else "unhealthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": VERSION,
        "checks": checks
    }
```

#### Monitoring and Alerting
```yaml
# prometheus/alerts.yml
groups:
- name: solidity-security-alerts
  rules:
  - alert: ServiceDown
    expr: up{job=~"solidity-security-.*"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Service {{ $labels.job }} is down"
      description: "Service {{ $labels.job }} has been down for more than 1 minute"

  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.job }}"

  - alert: DatabaseConnectionFailure
    expr: database_connections_failed_total > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Database connection failures detected"
      description: "{{ $value }} database connection failures in the last minute"

  - alert: SharedLibraryFallback
    expr: shared_library_rust_unavailable == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Shared library using fallback implementation"
      description: "Rust acceleration unavailable, using slower fallback"
```

## Automation Scripts and Tools

### Development Automation

#### Pre-commit Hooks
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-json
      - id: check-yaml

  - repo: https://github.com/psf/black
    rev: 23.9.1
    hooks:
      - id: black
        files: ^solidity-security-.*/.*\.py$

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
        files: ^solidity-security-.*/.*\.py$

  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        files: ^solidity-security-.*/.*\.py$

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.52.0
    hooks:
      - id: eslint
        files: ^solidity-security-.*/.*\.(ts|tsx|js|jsx)$
        additional_dependencies:
          - eslint@8.52.0
          - typescript@5.2.2

  - repo: https://github.com/doublify/pre-commit-rust
    rev: v1.0
    hooks:
      - id: fmt
        files: ^solidity-security-contract-parser/.*\.rs$
      - id: clippy
        files: ^solidity-security-contract-parser/.*\.rs$
```

#### Development Setup Automation
```bash
#!/bin/bash
# scripts/dev-setup.sh
set -e

echo "Setting up Solidity Security Platform development environment..."

# Check prerequisites
command -v python3 >/dev/null 2>&1 || { echo "Python 3 is required"; exit 1; }
command -v node >/dev/null 2>&1 || { echo "Node.js is required"; exit 1; }
command -v cargo >/dev/null 2>&1 || { echo "Rust is required"; exit 1; }

# Build shared library first
echo "Building shared library..."
cd solidity-security-shared
make install-deps
make build
make test
cd ..

# Set up Python services
for service in api-service tool-integration intelligence-engine orchestration data-service notification; do
  echo "Setting up solidity-security-$service..."
  cd "solidity-security-$service"

  python3 -m venv venv
  source venv/bin/activate
  pip install -r requirements/dev.txt

  # Create .env from example
  if [ -f .env.example ]; then
    cp .env.example .env
  fi

  cd ..
done

# Set up TypeScript services
for service in ui-core dashboard findings analysis; do
  echo "Setting up solidity-security-$service..."
  cd "solidity-security-$service"

  npm install

  # Create .env from example
  if [ -f .env.example ]; then
    cp .env.example .env.local
  fi

  cd ..
done

# Set up Rust service
echo "Setting up solidity-security-contract-parser..."
cd solidity-security-contract-parser
cargo build
cargo test

# Create .env from example
if [ -f .env.example ]; then
  cp .env.example .env
fi

cd ..

# Install pre-commit hooks
pip install pre-commit
pre-commit install

echo "Development environment setup complete!"
echo "Run 'scripts/start-dev-services.sh' to start all services in development mode."
```

### Maintenance and Operations

#### Database Migration Automation
```python
#!/usr/bin/env python3
# scripts/run_migrations.py
import subprocess
import sys
import os
from pathlib import Path

def run_migrations():
    """Run database migrations for all Python services."""

    services_with_db = [
        'solidity-security-api-service',
        'solidity-security-data-service',
        'solidity-security-intelligence-engine',
        'solidity-security-orchestration'
    ]

    for service in services_with_db:
        service_path = Path(service)
        if not service_path.exists():
            print(f"Warning: {service} directory not found")
            continue

        print(f"Running migrations for {service}...")

        # Change to service directory
        os.chdir(service_path)

        try:
            # Run Alembic migrations
            result = subprocess.run(
                ['alembic', 'upgrade', 'head'],
                capture_output=True,
                text=True,
                check=True
            )
            print(f"✅ {service} migrations completed successfully")
        except subprocess.CalledProcessError as e:
            print(f"❌ {service} migrations failed:")
            print(e.stdout)
            print(e.stderr)
            sys.exit(1)
        finally:
            # Change back to root directory
            os.chdir('..')

if __name__ == "__main__":
    run_migrations()
```

## Best Practices and Guidelines

### CI/CD Best Practices

1. **Fast Feedback**: Optimize pipeline for speed while maintaining quality
2. **Fail Fast**: Run fastest tests first to provide immediate feedback
3. **Parallel Execution**: Use matrix strategies for concurrent builds
4. **Caching**: Implement effective caching for dependencies and build artifacts
5. **Security**: Scan for vulnerabilities and secrets in every build

### Quality Gates

1. **Code Coverage**: Maintain >80% coverage for critical components
2. **Performance**: Monitor and prevent performance regressions
3. **Security**: Block deployments with high/critical security issues
4. **Dependencies**: Keep dependencies up to date and secure

### Deployment Strategies

1. **Blue-Green Deployments**: Zero-downtime deployments with quick rollback
2. **Health Checks**: Comprehensive health monitoring for deployment verification
3. **Gradual Rollouts**: Progressive deployment to minimize risk
4. **Rollback Capability**: Quick rollback mechanisms for failed deployments

This comprehensive CI/CD and automation guide ensures reliable, secure, and efficient software delivery across all services while maintaining high quality standards and enabling rapid development iteration.