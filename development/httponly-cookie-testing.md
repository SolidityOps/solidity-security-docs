# Testing Guide

**Version:** 1.0.0
**Last Updated:** October 9, 2025

## Table of Contents

1. [Overview](#overview)
2. [Test Infrastructure](#test-infrastructure)
3. [Running Tests](#running-tests)
4. [Test Types](#test-types)
5. [Writing Tests](#writing-tests)
6. [Coverage Reports](#coverage-reports)
7. [CI/CD Integration](#cicd-integration)
8. [Troubleshooting](#troubleshooting)

---

## Overview

The Solidity Security API Service has a comprehensive automated testing suite that ensures code quality, security, and functionality. Tests are organized into multiple categories and run automatically in CI/CD pipelines.

### Testing Stack

- **Framework:** pytest 7.4.3+
- **Coverage:** pytest-cov with 75% minimum threshold
- **HTTP Testing:** requests library for HttpOnly cookie support
- **Fixtures:** Shared fixtures in `tests/conftest.py`
- **Markers:** Custom markers for test categorization
- **CI/CD:** GitHub Actions with PostgreSQL and Redis services

---

## Test Infrastructure

### Directory Structure

```
tests/
├── __init__.py                         # Package marker
├── conftest.py                         # Shared fixtures and configuration
├── integration/                        # Integration tests
│   ├── __init__.py
│   ├── test_auth_api.py               # Authentication API tests
│   ├── test_contracts_api.py          # Contracts API tests
│   ├── test_e2e_scan_flow.py          # End-to-end scan workflow
│   └── test_multifile_upload.py       # Multi-file upload tests
└── security/                           # Security-focused tests
    ├── __init__.py
    ├── test_rate_limiting.py          # Rate limiting tests
    └── test_network_isolation.py      # Network isolation tests
```

### Configuration Files

**pyproject.toml** - Main pytest configuration:
```toml
[tool.pytest.ini_options]
minversion = "7.0"
addopts = "-ra -q --strict-markers --strict-config --cov=src --cov-report=html --cov-report=term-missing --cov-report=json --cov-branch --cov-fail-under=75"
testpaths = ["tests"]
markers = [
    "integration: Integration tests that require running services",
    "unit: Unit tests",
    "e2e: End-to-end tests",
    "security: Security-related tests",
    "multifile: Multi-file upload tests",
    "slow: Slow-running tests",
]
asyncio_mode = "auto"
timeout = 300
```

**requirements/test.txt** - Test dependencies:
```
pytest>=7.4.3,<8.0.0
pytest-asyncio>=0.21.1,<1.0.0
pytest-cov>=4.1.0,<5.0.0
pytest-mock>=3.12.0,<4.0.0
pytest-xdist>=3.5.0,<4.0.0
requests>=2.31.0,<3.0.0
faker>=20.1.0,<21.0.0
```

---

## Running Tests

### Prerequisites

1. **Install dependencies:**
   ```bash
   pip install -r requirements/test.txt
   ```

2. **Set up environment variables:**
   ```bash
   cp .env.example .env.test
   # Edit .env.test with test database credentials
   ```

3. **Start required services (PostgreSQL, Redis):**
   ```bash
   # Using Docker Compose
   docker-compose up -d postgres redis

   # Or using Kubernetes
   kubectl port-forward -n postgresql-local svc/postgresql 5432:5432
   kubectl port-forward -n redis-local svc/redis-master 6379:6379
   ```

### Running All Tests

```bash
# Run all tests with coverage
pytest

# Run with verbose output
pytest -v

# Run with detailed output
pytest -vv

# Run tests in parallel (faster)
pytest -n auto
```

### Running Specific Test Categories

```bash
# Unit tests only
pytest -m unit

# Integration tests only
pytest -m integration

# E2E tests only
pytest -m e2e

# Security tests only
pytest -m security

# Multi-file upload tests only
pytest -m multifile

# Exclude slow tests
pytest -m "not slow"
```

### Running Specific Test Files

```bash
# Run authentication tests
pytest tests/integration/test_auth_api.py -v

# Run contract API tests
pytest tests/integration/test_contracts_api.py -v

# Run E2E workflow tests
pytest tests/integration/test_e2e_scan_flow.py -v

# Run multi-file upload tests
pytest tests/integration/test_multifile_upload.py -v
```

### Running Specific Tests

```bash
# Run a specific test function
pytest tests/integration/test_auth_api.py::TestAuthenticationAPI::test_register_success -v

# Run tests matching a pattern
pytest -k "test_register" -v

# Run tests matching multiple patterns
pytest -k "test_register or test_login" -v
```

---

## Test Types

### 1. Unit Tests (`@pytest.mark.unit`)

**Purpose:** Test individual functions and classes in isolation.

**Characteristics:**
- Fast execution (< 1 second per test)
- No external dependencies
- Mock external services
- High code coverage

**Example:**
```python
@pytest.mark.unit
def test_password_hashing():
    """Test password hashing utility"""
    password = "TestPassword123!"
    hashed = hash_password(password)
    assert verify_password(password, hashed)
```

### 2. Integration Tests (`@pytest.mark.integration`)

**Purpose:** Test API endpoints with real database and services.

**Characteristics:**
- Require PostgreSQL and Redis
- Use HttpOnly cookie authentication
- Test full request/response cycle
- Validate data persistence

**Example:**
```python
@pytest.mark.integration
class TestAuthenticationAPI:
    def test_register_success(self, http_session, test_user_credentials):
        """Test successful user registration"""
        response = http_session.post(
            f"{API_BASE_URL}/auth/register",
            json=test_user_credentials,
        )
        assert response.status_code == 201
        assert "access_token" in http_session.cookies
```

### 3. End-to-End Tests (`@pytest.mark.e2e`)

**Purpose:** Test complete user workflows from start to finish.

**Characteristics:**
- Test multiple API calls in sequence
- Verify business logic flows
- Validate state transitions
- May be slow (> 5 seconds)

**Example:**
```python
@pytest.mark.e2e
@pytest.mark.slow
def test_complete_scan_workflow(authenticated_session):
    """Test: Upload contract → Create scan → Wait for results → Get vulnerabilities"""
    # Upload contract
    contract_response = authenticated_session.post(...)
    contract_id = contract_response.json()["id"]

    # Create scan
    scan_response = authenticated_session.post(...)
    scan_id = scan_response.json()["id"]

    # Wait for completion
    # ...

    # Get vulnerabilities
    # ...
```

### 4. Security Tests (`@pytest.mark.security`)

**Purpose:** Validate security controls and authentication.

**Characteristics:**
- Test authentication/authorization
- Validate OWASP compliance
- Test rate limiting
- Verify HttpOnly cookies

**Example:**
```python
@pytest.mark.security
def test_unauthorized_access():
    """Test that endpoints require authentication"""
    unauthenticated_session = requests.Session()
    response = unauthenticated_session.get(f"{API_BASE_URL}/contracts")
    assert response.status_code == 401
```

### 5. Multi-File Tests (`@pytest.mark.multifile`)

**Purpose:** Test ZIP and TAR.GZ archive uploads.

**Characteristics:**
- Test file extraction
- Validate multi-file contracts
- Test nested directories
- Verify file metadata

**Example:**
```python
@pytest.mark.multifile
def test_upload_zip_archive(authenticated_session):
    """Test uploading ZIP archive with multiple Solidity files"""
    zip_data = create_zip_archive()
    files = {'file': ('project.zip', io.BytesIO(zip_data), 'application/zip')}
    response = authenticated_session.post(f"{API_BASE_URL}/contracts/upload", files=files)
    assert response.status_code == 201
    assert response.json()["is_multi_file"] is True
```

---

## Writing Tests

### HttpOnly Cookie Authentication

All integration and E2E tests use HttpOnly cookies for authentication. **Never** use Bearer tokens in tests.

#### ✅ Correct Pattern:

```python
def test_with_authentication(authenticated_session: requests.Session):
    """Use authenticated_session fixture - cookies handled automatically"""
    response = authenticated_session.get(f"{API_BASE_URL}/contracts")
    assert response.status_code == 200
```

#### ❌ Incorrect Pattern:

```python
# DON'T DO THIS - Bearer tokens are deprecated
def test_with_old_auth():
    token = get_auth_token()
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(f"{API_BASE_URL}/contracts", headers=headers)
```

### Using Shared Fixtures

The `tests/conftest.py` file provides reusable fixtures:

```python
# HTTP session with cookie support
def test_something(http_session: requests.Session):
    """Use http_session for unauthenticated requests"""
    response = http_session.get(f"{API_BASE_URL}/health")
    assert response.status_code == 200

# Authenticated HTTP session
def test_authenticated(authenticated_session: requests.Session):
    """Use authenticated_session for protected endpoints"""
    response = authenticated_session.get(f"{API_BASE_URL}/contracts")
    assert response.status_code == 200

# Test user credentials
def test_with_credentials(test_user_credentials: dict):
    """Use test_user_credentials for unique test users"""
    email = test_user_credentials["email"]
    password = test_user_credentials["password"]

# Sample contracts
def test_with_contract(sample_solidity_contract: str):
    """Use sample_solidity_contract for testing"""
    # Contract source code provided
```

### Test Naming Conventions

```python
# Format: test_<action>_<condition>
def test_register_success():              # Good
def test_login_invalid_credentials():    # Good
def test_create_contract_without_auth(): # Good

def test1():                              # Bad
def my_test():                            # Bad
```

### Assertions Best Practices

```python
# Use descriptive assertion messages
assert response.status_code == 201, f"Expected 201, got {response.status_code}: {response.text}"

# Test multiple conditions
data = response.json()
assert "id" in data
assert "email" in data
assert data["email"] == expected_email

# Validate security properties
assert "access_token" not in data, "Tokens should not be in response body"
assert "access_token" in session.cookies, "Access token cookie not set"
```

### Test Organization

```python
@pytest.mark.integration
@pytest.mark.security
class TestAuthenticationAPI:
    """Group related tests in classes"""

    def test_register_success(self, http_session):
        """Test successful registration"""
        pass

    def test_register_duplicate_email(self, http_session):
        """Test duplicate email fails"""
        pass

    def test_login_success(self, http_session):
        """Test successful login"""
        pass
```

---

## Coverage Reports

### Minimum Coverage Requirement

**75% code coverage is mandatory** for all pull requests.

### Generating Coverage Reports

```bash
# Run tests with coverage
pytest --cov=src --cov-report=html --cov-report=term-missing

# View HTML report
open htmlcov/index.html

# View terminal report
pytest --cov=src --cov-report=term-missing

# Generate JSON report
pytest --cov=src --cov-report=json

# Check if coverage meets threshold
pytest --cov=src --cov-fail-under=75
```

### Coverage Report Formats

1. **Terminal Report:**
   ```
   Name                        Stmts   Miss  Cover   Missing
   ---------------------------------------------------------
   src/main.py                    45      3    93%   23-25
   src/domain/entities/user.py    67      5    93%   78-82
   src/application/handlers.py   123      8    93%   145-152
   ---------------------------------------------------------
   TOTAL                         856     64    93%
   ```

2. **HTML Report:**
   - Interactive browsable report
   - Line-by-line coverage visualization
   - Missing coverage highlighted in red
   - Located in `htmlcov/index.html`

3. **JSON Report:**
   - Machine-readable format
   - Used by CI/CD and coverage tools
   - Located in `coverage.json`

### Improving Coverage

```python
# Identify untested code
pytest --cov=src --cov-report=term-missing

# Focus on specific modules
pytest --cov=src.domain --cov-report=term-missing

# Test only missing lines
pytest tests/test_specific.py --cov=src.module --cov-report=term-missing
```

---

## CI/CD Integration

### GitHub Actions Workflow

Tests run automatically on:
- Push to `main` or `develop` branches
- Pull requests to `main` or `develop`
- Feature branch pushes

**Workflow file:** `.github/workflows/test.yml`

### CI/CD Jobs

1. **Test Job:**
   - Runs unit, integration, and E2E tests
   - Generates coverage reports
   - Uploads to Codecov
   - Fails if coverage < 75%

2. **Quality Job:**
   - Code formatting (Black)
   - Linting (Ruff)
   - Type checking (mypy)

3. **Security Job:**
   - Dependency vulnerabilities (safety)
   - Security linting (bandit)

4. **Build Job:**
   - Builds Docker image with semantic versioning
   - Scans image with Trivy
   - Pushes to GitHub Container Registry

### Local CI/CD Simulation

```bash
# Run the same checks as CI/CD
./scripts/ci-check.sh

# Or manually:
pytest --cov=src --cov-fail-under=75
black --check src/ tests/
ruff check src/ tests/
mypy src/
```

### Viewing CI/CD Results

1. **GitHub Actions Tab:**
   - View workflow runs
   - Check job logs
   - Download artifacts

2. **Pull Request Checks:**
   - All checks must pass before merge
   - Coverage report visible in PR comments

3. **Codecov Dashboard:**
   - Detailed coverage analysis
   - Coverage trends over time
   - File-by-file coverage

---

## Troubleshooting

### Common Issues

#### 1. Database Connection Errors

**Error:** `sqlalchemy.exc.OperationalError: could not connect to server`

**Solution:**
```bash
# Check PostgreSQL is running
kubectl get pods -n postgresql-local

# Port forward PostgreSQL
kubectl port-forward -n postgresql-local svc/postgresql 5432:5432

# Verify connection
psql -h localhost -U postgres -d solidity_security_test
```

#### 2. Redis Connection Errors

**Error:** `redis.exceptions.ConnectionError: Error connecting to Redis`

**Solution:**
```bash
# Check Redis is running
kubectl get pods -n redis-local

# Port forward Redis
kubectl port-forward -n redis-local svc/redis-master 6379:6379

# Test connection
redis-cli ping
```

#### 3. HttpOnly Cookie Not Set

**Error:** `AssertionError: Access token cookie not set`

**Solution:**
- Verify API is running on correct port (8001)
- Check `.env.test` has correct API_BASE_URL
- Ensure cookies are not being cleared between requests
- Use `requests.Session()` instead of individual requests

#### 4. Import Errors

**Error:** `ModuleNotFoundError: No module named 'src'`

**Solution:**
```bash
# Set PYTHONPATH
export PYTHONPATH="${PYTHONPATH}:$(pwd)/src"

# Or install package in editable mode
pip install -e .
```

#### 5. Coverage Below Threshold

**Error:** `FAILED Required test coverage of 75% not reached`

**Solution:**
```bash
# Identify untested code
pytest --cov=src --cov-report=term-missing

# Write tests for missing coverage
# Focus on critical paths first
```

### Debugging Tests

```bash
# Run with detailed output
pytest -vvs

# Drop into debugger on failure
pytest --pdb

# Stop at first failure
pytest -x

# Print output even on success
pytest -s

# Run specific test with debugging
pytest tests/integration/test_auth_api.py::test_register_success -vvs --pdb
```

### Performance Issues

```bash
# Run tests in parallel
pytest -n auto

# Exclude slow tests during development
pytest -m "not slow"

# Profile test execution time
pytest --durations=10
```

---

## Best Practices

### DO:
✅ Use `authenticated_session` fixture for HttpOnly cookies
✅ Test both success and failure scenarios
✅ Use descriptive test names and docstrings
✅ Group related tests in classes
✅ Use appropriate pytest markers
✅ Write tests before fixing bugs (TDD)
✅ Maintain > 75% code coverage
✅ Keep tests fast and isolated

### DON'T:
❌ Use Bearer tokens (deprecated)
❌ Hardcode credentials or secrets
❌ Share state between tests
❌ Skip security tests
❌ Commit failing tests
❌ Test implementation details
❌ Write tests without assertions

---

## Resources

- **pytest Documentation:** https://docs.pytest.org/
- **pytest-cov Documentation:** https://pytest-cov.readthedocs.io/
- **OWASP Testing Guide:** https://owasp.org/www-project-web-security-testing-guide/
- **Coverage.py:** https://coverage.readthedocs.io/

---

## Getting Help

- **Issues:** Report testing issues in GitHub Issues
- **Questions:** Ask in team Slack channel #testing
- **Documentation:** Check `/docs` directory for more guides

---

**Remember:** Tests are documentation. Write tests that clearly express intent and expected behavior.
