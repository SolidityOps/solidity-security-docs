# Docker Image Standards

**Version:** 1.0.0
**Last Updated:** October 9, 2025
**Status:** Active

## Table of Contents

1. [Overview](#overview)
2. [Mandatory Requirements](#mandatory-requirements)
3. [Semantic Versioning](#semantic-versioning)
4. [Dockerfile Standards](#dockerfile-standards)
5. [Image Tagging Strategy](#image-tagging-strategy)
6. [Build Arguments](#build-arguments)
7. [Multi-Stage Builds](#multi-stage-builds)
8. [Security Requirements](#security-requirements)
9. [Build and Push Workflow](#build-and-push-workflow)
10. [Examples](#examples)

---

## Overview

This document defines mandatory standards for all Docker images in the Solidity Security Platform. These standards ensure:

- **Reproducible builds** through semantic versioning
- **Cache busting** to prevent stale image usage
- **Security** through proper image construction
- **Consistency** across all services
- **Traceability** with version labels

**All Docker images MUST comply with these standards without exception.**

---

## Mandatory Requirements

### 1. Dockerfile Requirement

**RULE:** Every service MUST have a Dockerfile in its repository root.

```
✅ CORRECT:
solidity-security-api-service/
├── Dockerfile
├── src/
├── requirements/
└── ...

❌ INCORRECT:
- Using Dockerfiles from other repositories
- Building images without Dockerfiles
- Dockerfiles in non-standard locations (unless documented)
```

### 2. Semantic Versioning Requirement

**RULE:** All images MUST use semantic versioning (semver) and MUST NOT use `latest` tag in production.

```
✅ CORRECT:
- myregistry.io/api-service:1.2.3
- myregistry.io/api-service:2.0.0-beta.1
- myregistry.io/api-service:1.5.7

❌ INCORRECT:
- myregistry.io/api-service:latest
- myregistry.io/api-service:v1
- myregistry.io/api-service:prod
```

**Why:** Using `latest` or non-versioned tags causes:
- **Cache issues** - Docker may use stale cached images
- **Deployment failures** - Unable to rollback to specific versions
- **Debugging nightmares** - Unknown which code version is running
- **Security risks** - Cannot track which vulnerabilities affect which deployments

---

## Semantic Versioning

### Version Format

All images MUST follow [Semantic Versioning 2.0.0](https://semver.org/):

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]
```

**Components:**
- **MAJOR**: Incompatible API changes (e.g., 1.x.x → 2.0.0)
- **MINOR**: Backward-compatible functionality (e.g., 1.2.x → 1.3.0)
- **PATCH**: Backward-compatible bug fixes (e.g., 1.2.3 → 1.2.4)
- **PRERELEASE** (optional): alpha, beta, rc versions (e.g., 2.0.0-beta.1)
- **BUILD** (optional): Build metadata (e.g., 1.2.3+20250109)

### Version Source

**RULE:** Version numbers MUST be stored in a `VERSION` file at the repository root.

```bash
# Example: VERSION file
0.3.6
```

**Rationale:**
- Single source of truth
- Easy to update programmatically
- CI/CD can read version automatically
- Consistent across all build tools

### When to Increment Versions

| Change Type | Version Component | Example |
|-------------|-------------------|---------|
| Breaking API changes | MAJOR | 1.5.3 → 2.0.0 |
| New features (backward compatible) | MINOR | 1.5.3 → 1.6.0 |
| Bug fixes (backward compatible) | PATCH | 1.5.3 → 1.5.4 |
| Security patches | PATCH | 1.5.3 → 1.5.4 |
| Documentation only | No change | 1.5.3 → 1.5.3 |
| Development/testing | PRERELEASE | 1.5.3 → 1.6.0-beta.1 |

---

## Dockerfile Standards

### Required Build Arguments

All Dockerfiles MUST include these build arguments:

```dockerfile
# Service identification
ARG SERVICE_NAME=your-service-name
ARG SERVICE_VERSION
ARG BUILD_DATE
ARG VCS_REF

# Language/runtime version
ARG PYTHON_VERSION=3.13
# OR
ARG NODE_VERSION=20
# OR
ARG RUST_VERSION=1.90
```

### Required Labels

All images MUST include these OCI-compliant labels:

```dockerfile
LABEL maintainer="Solidity Security Team <team@soliditysecurity.com>" \
      org.label-schema.name="${SERVICE_NAME}" \
      org.label-schema.version="${SERVICE_VERSION}" \
      org.label-schema.description="Service description here" \
      org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.schema-version="1.0"
```

**Why labels matter:**
- Inspect running containers to determine exact version
- Automated vulnerability scanning
- Compliance and audit trails
- Debugging production issues

### Multi-Stage Build Template

All services SHOULD use multi-stage builds for optimization:

```dockerfile
# Stage 1: Builder
FROM python:3.13-slim as builder

ARG SERVICE_VERSION
ARG BUILD_DATE
ARG VCS_REF

# Install build dependencies
RUN apt-get update && apt-get install -y gcc make \
    && rm -rf /var/lib/apt/lists/*

# Install application dependencies
COPY requirements/ requirements/
RUN pip install --user --no-cache-dir -r requirements/base.txt

# Stage 2: Test Builder (optional but recommended)
FROM builder as test-builder

# Install test dependencies
RUN pip install --user --no-cache-dir -r requirements/test.txt

# Stage 3: Runtime
FROM python:3.13-slim as runtime

ARG SERVICE_NAME
ARG SERVICE_VERSION
ARG BUILD_DATE
ARG VCS_REF

# Add labels
LABEL maintainer="Solidity Security Team <team@soliditysecurity.com>" \
      org.label-schema.name="${SERVICE_NAME}" \
      org.label-schema.version="${SERVICE_VERSION}" \
      org.label-schema.description="Service description" \
      org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.schema-version="1.0"

# Copy from builder
COPY --from=builder /root/.local /home/appuser/.local

# Application setup
WORKDIR /app
COPY . .

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "src.main:app"]
```

---

## Image Tagging Strategy

### Production Tags

**Format:** `<registry>/<service>:<version>`

```bash
# Production images
ghcr.io/org/api-service:1.2.3
ghcr.io/org/api-service:2.0.0
ghcr.io/org/api-service:1.5.7

# Pre-release images
ghcr.io/org/api-service:2.0.0-beta.1
ghcr.io/org/api-service:1.6.0-rc.2
```

### Development/Testing Tags

**Format:** `<registry>/<service>:<version>-<environment>`

```bash
# Development
ghcr.io/org/api-service:1.2.3-dev
ghcr.io/org/api-service:1.2.4-dev

# Staging
ghcr.io/org/api-service:1.2.3-staging

# Feature branches (include branch name)
ghcr.io/org/api-service:1.2.3-feature-auth-fix
```

### Additional Tags (Optional)

```bash
# Git commit SHA (for exact traceability)
ghcr.io/org/api-service:1.2.3-sha-a1b2c3d

# Build number (CI/CD tracking)
ghcr.io/org/api-service:1.2.3-build-456
```

### Never Use These Tags in Production

```bash
❌ ghcr.io/org/api-service:latest
❌ ghcr.io/org/api-service:prod
❌ ghcr.io/org/api-service:stable
❌ ghcr.io/org/api-service:v1
❌ ghcr.io/org/api-service:main
```

---

## Build Arguments

### Passing Version to Build

**RULE:** Always pass `SERVICE_VERSION` as a build argument from the `VERSION` file.

```bash
# Read version from VERSION file
VERSION=$(cat VERSION | tr -d '\n')

# Build with semantic version
docker build \
  --build-arg SERVICE_VERSION=${VERSION} \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg VCS_REF=$(git rev-parse --short HEAD) \
  -t myregistry.io/api-service:${VERSION} \
  .
```

### Automated Build Script Example

Create a `scripts/build-image.sh` in each repository:

```bash
#!/bin/bash
set -e

# Read version from VERSION file
VERSION=$(cat VERSION | tr -d '\n')
SERVICE_NAME=$(basename $(pwd))
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
VCS_REF=$(git rev-parse --short HEAD)
REGISTRY=${DOCKER_REGISTRY:-"ghcr.io/soliditysecurity"}

echo "Building ${SERVICE_NAME}:${VERSION}"

docker build \
  --build-arg SERVICE_NAME=${SERVICE_NAME} \
  --build-arg SERVICE_VERSION=${VERSION} \
  --build-arg BUILD_DATE=${BUILD_DATE} \
  --build-arg VCS_REF=${VCS_REF} \
  -t ${REGISTRY}/${SERVICE_NAME}:${VERSION} \
  -f Dockerfile \
  .

echo "✅ Built: ${REGISTRY}/${SERVICE_NAME}:${VERSION}"
```

---

## Multi-Stage Builds

### Benefits

1. **Smaller images** - Runtime image excludes build tools
2. **Security** - Reduced attack surface
3. **Faster deployments** - Less data to transfer
4. **Test isolation** - Test dependencies separate from production

### Standard Stages

```dockerfile
# Stage 1: builder - Install dependencies
FROM <base> as builder

# Stage 2: test-builder - Add test dependencies (optional)
FROM builder as test-builder

# Stage 3: runtime - Final production image
FROM <base> as runtime
COPY --from=builder /app /app
```

### Testing in CI/CD

```bash
# Build test stage
docker build --target test-builder -t api-service:test .

# Run tests
docker run --rm api-service:test pytest

# Build production stage
docker build --target runtime -t api-service:${VERSION} .
```

---

## Security Requirements

### 1. Non-Root User

**RULE:** All containers MUST run as non-root user.

```dockerfile
# Create non-root user
RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash --create-home appuser

# Switch to non-root user
USER appuser
```

### 2. No Secrets in Images

**RULE:** Never bake secrets into images.

```dockerfile
❌ WRONG:
ENV DATABASE_PASSWORD=mysecret123

✅ CORRECT:
# Use environment variables at runtime
# Use Kubernetes secrets
# Use vault integration
```

### 3. Minimal Base Images

**RULE:** Use minimal base images when possible.

```dockerfile
✅ PREFERRED:
FROM python:3.13-slim
FROM node:20-alpine
FROM rust:1.90-slim

❌ AVOID:
FROM python:3.13        # Full image (larger attack surface)
FROM ubuntu:latest      # Not minimal
```

### 4. Vulnerability Scanning

**RULE:** All images MUST pass vulnerability scanning before production deployment.

```bash
# Scan image for vulnerabilities
docker scan myregistry.io/api-service:1.2.3

# Or use trivy
trivy image myregistry.io/api-service:1.2.3
```

---

## Build and Push Workflow

### Local Development Build

```bash
# 1. Update VERSION file
echo "1.2.4" > VERSION

# 2. Build image
./scripts/build-image.sh

# 3. Test image locally
docker run --rm -p 8000:8000 myregistry.io/api-service:1.2.4

# 4. Push to registry
docker push myregistry.io/api-service:1.2.4
```

### CI/CD Build (GitHub Actions Example)

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Read version
        id: version
        run: echo "VERSION=$(cat VERSION | tr -d '\n')" >> $GITHUB_OUTPUT

      - name: Build image
        run: |
          docker build \
            --build-arg SERVICE_VERSION=${{ steps.version.outputs.VERSION }} \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            --build-arg VCS_REF=${{ github.sha }} \
            -t ghcr.io/org/api-service:${{ steps.version.outputs.VERSION }} \
            .

      - name: Push image
        run: docker push ghcr.io/org/api-service:${{ steps.version.outputs.VERSION }}
```

---

## Examples

### Example 1: Python FastAPI Service

```dockerfile
# Multi-stage build for Python FastAPI service
FROM python:3.13-slim as builder

ARG SERVICE_VERSION
ARG PYTHON_VERSION=3.13

RUN apt-get update && apt-get install -y gcc make curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements/ requirements/
RUN pip install --user --no-cache-dir -r requirements/base.txt

# Test stage
FROM builder as test-builder
RUN pip install --user --no-cache-dir -r requirements/test.txt

# Runtime
FROM python:3.13-slim as runtime

ARG SERVICE_NAME=api-service
ARG SERVICE_VERSION
ARG BUILD_DATE
ARG VCS_REF

LABEL maintainer="Solidity Security Team <team@soliditysecurity.com>" \
      org.label-schema.name="${SERVICE_NAME}" \
      org.label-schema.version="${SERVICE_VERSION}" \
      org.label-schema.description="FastAPI gateway service" \
      org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.schema-version="1.0"

RUN apt-get update && apt-get install -y curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash --create-home appuser

ENV PYTHONUNBUFFERED=1 \
    PATH="/home/appuser/.local/bin:$PATH"

WORKDIR /app
COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Example 2: Node.js TypeScript Service

```dockerfile
FROM node:20-alpine as builder

ARG SERVICE_VERSION
ARG NODE_VERSION=20

WORKDIR /app
COPY package*.json ./
RUN npm ci --production=false

COPY . .
RUN npm run build

# Test stage
FROM builder as test-builder
RUN npm run test

# Runtime
FROM node:20-alpine as runtime

ARG SERVICE_NAME=notification-service
ARG SERVICE_VERSION
ARG BUILD_DATE
ARG VCS_REF

LABEL maintainer="Solidity Security Team <team@soliditysecurity.com>" \
      org.label-schema.name="${SERVICE_NAME}" \
      org.label-schema.version="${SERVICE_VERSION}" \
      org.label-schema.description="Notification service" \
      org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.schema-version="1.0"

RUN apk add --no-cache curl

RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/dist ./dist
COPY --from=builder --chown=appuser:appuser /app/node_modules ./node_modules
COPY --chown=appuser:appuser package*.json ./

USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "dist/main.js"]
```

### Example 3: Rust Service

```dockerfile
FROM rust:1.90-slim as builder

ARG SERVICE_VERSION
ARG RUST_VERSION=1.90

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src/ src/

RUN cargo build --release

# Runtime
FROM debian:bookworm-slim as runtime

ARG SERVICE_NAME=contract-parser
ARG SERVICE_VERSION
ARG BUILD_DATE
ARG VCS_REF

LABEL maintainer="Solidity Security Team <team@soliditysecurity.com>" \
      org.label-schema.name="${SERVICE_NAME}" \
      org.label-schema.version="${SERVICE_VERSION}" \
      org.label-schema.description="Contract parser service" \
      org.label-schema.build-date="${BUILD_DATE}" \
      org.label-schema.vcs-ref="${VCS_REF}" \
      org.label-schema.schema-version="1.0"

RUN apt-get update && apt-get install -y curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash --create-home appuser

WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/target/release/contract-parser .

USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s CMD curl -f http://localhost:8080/health || exit 1

CMD ["./contract-parser"]
```

---

## Compliance Checklist

Before deploying any Docker image to production, verify:

- [ ] Dockerfile exists in repository root
- [ ] VERSION file exists with semantic version
- [ ] Dockerfile includes all required build arguments
- [ ] Dockerfile includes all required labels
- [ ] Image uses semantic versioning tag (not `latest`)
- [ ] Multi-stage build is used (if applicable)
- [ ] Container runs as non-root user
- [ ] No secrets baked into image
- [ ] Minimal base image used
- [ ] Vulnerability scan passed
- [ ] Health check defined
- [ ] Build script created (scripts/build-image.sh)
- [ ] CI/CD workflow includes version from VERSION file

---

## Standards Reference

This document references:
- [Semantic Versioning 2.0.0](https://semver.org/)
- [OCI Image Spec](https://github.com/opencontainers/image-spec)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

---

## Document History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2025-10-09 | Initial standards document | Solidity Security Team |

---

## Questions or Issues?

Contact the DevOps team or create an issue in the `solidity-security-docs` repository.

**Remember: These are MANDATORY standards. Non-compliance will block deployment to production.**
