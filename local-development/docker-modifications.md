# Docker Image Build Modifications

> **⚠️ LOCAL DEVELOPMENT ONLY - These modifications are NOT for production use**

## Overview

This document details all Docker image build modifications made to support local development, including dependency management, shared library integration, and build optimizations.

## Shared Library Integration

### Problem Addressed
Service Docker builds were failing because the `solidity-security-shared>=0.1.0` dependency was not available in standard package repositories.

### Solution Implemented

#### Modified Dockerfile Structure
```dockerfile
# Before (failing)
COPY requirements/ requirements/
RUN pip install --user --no-cache-dir -r requirements/base.txt -r requirements/dev.txt

# After (working)
COPY requirements/ requirements/
COPY solidity_security_shared-0.1.0-py3-none-any.whl .  # ⚠️ LOCAL DEV ONLY

# Install shared library first, then other dependencies
RUN pip install --user --no-cache-dir solidity_security_shared-0.1.0-py3-none-any.whl
RUN pip install --user --no-cache-dir -r requirements/base.txt
```

#### Files Added to Each Service
```bash
# Wheel copied to each Python service directory
solidity-security-api-service/solidity_security_shared-0.1.0-py3-none-any.whl
solidity-security-data-service/solidity_security_shared-0.1.0-py3-none-any.whl
solidity-security-intelligence-engine/solidity_security_shared-0.1.0-py3-none-any.whl
solidity-security-orchestration/solidity_security_shared-0.1.0-py3-none-any.whl
solidity-security-tool-integration/solidity_security_shared-0.1.0-py3-none-any.whl
```

**⚠️ Production Note**: Production builds should use proper package management (PyPI, private registry, or pip-installable URLs).

## Development Dependencies Exclusion

### Problem Addressed
Service builds were failing on missing development-only packages like `httpx-mock<1.0.0,>=0.10.1` that aren't available in standard repositories.

### Solution Applied

#### Dockerfile Modification
```dockerfile
# Before (failing in production context)
RUN pip install --user --no-cache-dir -r requirements/base.txt -r requirements/dev.txt

# After (production-ready)
RUN pip install --user --no-cache-dir -r requirements/base.txt
```

#### Rationale
- Production Docker images don't need development dependencies
- Testing tools like `httpx-mock`, `pytest-cov`, etc., are not required in runtime
- Reduces image size and potential security vulnerabilities
- Focuses on production runtime requirements only

**✅ Production Alignment**: This change actually makes the build MORE production-ready.

## Permission and Directory Fixes

### Problem Addressed
Docker builds were failing with permission denied errors when creating application directories after switching to non-root user.

### Original Issue
```dockerfile
# Problematic order
COPY --chown=appuser:appuser . .
USER appuser
RUN mkdir -p /app/logs /app/data  # ❌ Fails - non-root can't create in /app
```

### Solution Applied
```dockerfile
# Fixed order - create directories as root, then switch user
# Create necessary directories with proper permissions (as root)
RUN mkdir -p /app/logs /app/data && \
    chmod 755 /app/logs /app/data && \
    chown appuser:appuser /app/logs /app/data

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser
```

**✅ Production Compatible**: This fix maintains security best practices while ensuring functionality.

## Build Context Optimization

### .dockerignore Configuration

Ensured wheel files are included in build context:
```bash
# Verified .dockerignore doesn't exclude:
*.whl          # ❌ Don't ignore wheel files
requirements/  # ❌ Don't ignore requirements
```

### Build Context Size
Local wheel files add minimal size (~50KB each) to build context.

## Multi-Stage Build Preservation

### Maintained Production Best Practices

The multi-stage build structure was preserved:

```dockerfile
# Stage 1: Builder - Install dependencies
FROM python:3.11-slim as builder
# ... dependency installation

# Stage 2: Runtime - Create final image
FROM python:3.11-slim as runtime
# ... copy only runtime artifacts
COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
```

**Benefits Maintained:**
- Smaller final image size
- No build tools in production image
- Separated concerns between build and runtime

## Service-Specific Modifications

### API Service (`solidity-security-api-service`)

#### Modified Files
- `Dockerfile` - Added wheel installation and fixed directory permissions
- Added: `solidity_security_shared-0.1.0-py3-none-any.whl`

#### Verification
```bash
# Built successfully
docker build -t localhost:5000/solidity-security-api-service:dev .

# Pushed to local registry
docker push localhost:5000/solidity-security-api-service:dev

# Shared library verified in image
docker run --rm localhost:5000/solidity-security-api-service:dev python3 -c "import solidity_shared; print('Working')"
```

### Other Services (Prepared)

Similar modifications prepared for:
- `solidity-security-data-service`
- `solidity-security-intelligence-engine`
- `solidity-security-orchestration`
- `solidity-security-tool-integration`

Each has the wheel file copied and is ready for the same Dockerfile modifications.

## Production Migration Requirements

### Remove Local Wheel Installation

**❌ Remove from production Dockerfile:**
```dockerfile
COPY solidity_security_shared-0.1.0-py3-none-any.whl .
RUN pip install --user --no-cache-dir solidity_security_shared-0.1.0-py3-none-any.whl
```

**✅ Replace with proper package management:**
```dockerfile
# Option 1: PyPI (if published)
# RUN pip install --user --no-cache-dir solidity-security-shared>=0.1.0

# Option 2: Private registry
# RUN pip install --user --no-cache-dir \
#     --index-url https://pypi.company.com/simple \
#     solidity-security-shared>=0.1.0

# Option 3: Git dependency
# RUN pip install --user --no-cache-dir \
#     git+https://github.com/company/solidity-security-shared.git#subdirectory=python
```

### Restore Development Dependencies

**Production builds should include development tools:**
```dockerfile
# Restore for production (with proper dev dependencies)
RUN pip install --user --no-cache-dir -r requirements/base.txt -r requirements/dev.txt
```

### Remove Local Wheel Files

Before production deployment:
```bash
# Remove wheel files from service directories
find . -name "solidity_security_shared-*.whl" -delete
```

## Build Performance Optimizations

### Dependency Caching

The modification maintains Docker layer caching:
```dockerfile
# Copy requirements first (cached layer)
COPY requirements/ requirements/

# Copy wheel (new layer, but small)
COPY solidity_security_shared-0.1.0-py3-none-any.whl .

# Install dependencies (leverages cache if requirements unchanged)
RUN pip install --user --no-cache-dir solidity_security_shared-0.1.0-py3-none-any.whl
RUN pip install --user --no-cache-dir -r requirements/base.txt
```

### Build Time Impact
- Additional ~5 seconds for wheel installation
- Shared library dependency resolved without network calls
- Overall build reliability increased

## Registry Integration

### Local Registry Push

Successfully integrated with local registry:
```bash
# Build
docker build -t localhost:5000/solidity-security-api-service:dev .

# Push
docker push localhost:5000/solidity-security-api-service:dev

# Verify
curl http://localhost:5000/v2/_catalog
# {"repositories":["solidity-security-api-service"]}
```

### Image Tagging Strategy
- **Local Development**: `dev` tag
- **Production**: Should use semantic versioning (e.g., `v1.0.0`, `v1.0.1`)

## Security Considerations

### Maintained Security Practices ✅
- Multi-stage builds (no build tools in final image)
- Non-root user execution
- Minimal base image (python:3.11-slim)
- Proper file permissions
- No secrets in environment variables

### Local Development Relaxations ⚠️
- Development dependencies excluded (actually improves security)
- Local wheel files included in image (temporary for local dev)

## Troubleshooting

### Common Issues and Solutions

#### 1. Wheel File Not Found
```bash
# Error: Could not find solidity_security_shared-0.1.0-py3-none-any.whl
# Solution: Ensure wheel is in service directory
cp /path/to/wheel service-directory/
```

#### 2. Permission Denied on Directory Creation
```bash
# Error: mkdir: cannot create directory '/app/logs': Permission denied
# Solution: Create directories before USER directive (already implemented)
```

#### 3. Import Errors After Build
```bash
# Test inside container
docker run --rm -it localhost:5000/service:dev python3 -c "import solidity_shared"
```

## Rollback Procedures

### To Revert Modifications

1. **Remove wheel files:**
```bash
find . -name "solidity_security_shared-*.whl" -delete
```

2. **Revert Dockerfile changes:**
```dockerfile
# Remove these lines:
COPY solidity_security_shared-0.1.0-py3-none-any.whl .
RUN pip install --user --no-cache-dir solidity_security_shared-0.1.0-py3-none-any.whl

# Restore dev dependencies:
RUN pip install --user --no-cache-dir -r requirements/base.txt -r requirements/dev.txt
```

3. **Fix shared library dependency properly:**
   - Build proper Python package with maturin
   - Publish to PyPI or private registry
   - Update requirements.txt to use published version

## Documentation References

- [`shared-library-build.md`](./shared-library-build.md) - Details on wheel creation
- [`production-differences.md`](./production-differences.md) - Production requirements
- [`deployment-verification.md`](./deployment-verification.md) - Testing procedures

---

**Modification Date**: October 2, 2025
**Purpose**: Enable local development with shared library dependency
**Production Status**: ❌ Requires proper package management for production
**Rollback**: Available via documented procedures