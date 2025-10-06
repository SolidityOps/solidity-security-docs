# Deployment Notes - API Service

**Last Updated**: October 6, 2025
**Sprint**: Sprint 1, Task 1.11 - Dashboard Backend Implementation
**Status**: Development complete, authentication bug blocking production

---

## 🚨 CRITICAL ISSUES - MUST FIX BEFORE PRODUCTION

### 1. Bcrypt Password Hashing Failure ⚠️

**Severity**: CRITICAL
**Impact**: Complete authentication system failure - no users can register or login
**Status**: UNRESOLVED

**Symptoms**:
- All password registration attempts fail with error: `"password cannot be longer than 72 bytes, truncate manually if necessary"`
- Error occurs even with short passwords (8-11 characters)
- Error comes from `passlib.context.CryptContext` library, not application code

**Affected Code**:
- `src/infrastructure/security/password.py:9-11` - `hash_password()` function
- Uses: `passlib.context.CryptContext(schemes=["bcrypt"], deprecated="auto")`

**Test Evidence**:
```bash
# All of these failed with the same error:
curl -X POST /api/v1/auth/register -d '{"email": "test@test.com", "password": "password"}'     # 8 chars
curl -X POST /api/v1/auth/register -d '{"email": "test@test.com", "password": "testpass123"}'  # 11 chars
curl -X POST /api/v1/auth/register -d '{"email": "test@test.com", "password": "test123456"}'   # 10 chars
```

**Workaround Used for Testing**:
```sql
-- Manually inserted test user with pre-hashed password
INSERT INTO users (id, email, hashed_password, is_active)
VALUES (gen_random_uuid(), 'test@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/Lev.aOyG1xWOQD8Gm', true);
-- Password: "password"
```

**Suspected Root Causes**:
1. **Library version conflict**: passlib or bcrypt package version incompatibility
2. **Missing dependency**: bcrypt C extension not properly compiled
3. **Environment variable**: Possible BCRYPT_ROUNDS or similar misconfiguration

**Investigation Steps Needed**:
```bash
# Check installed versions
pip list | grep -E "(passlib|bcrypt)"

# Expected:
# passlib>=1.7.4
# bcrypt>=4.0.0

# Test bcrypt directly
python3 -c "import bcrypt; print(bcrypt.hashpw(b'password', bcrypt.gensalt()))"
```

**Recommended Fix**:
1. Verify passlib and bcrypt versions in `requirements/base.txt`
2. Pin specific working versions
3. Test password hashing in isolation before deployment
4. Add unit tests for password hashing (currently missing)

---

## ⚠️ HIGH PRIORITY - PRODUCTION BLOCKERS

### 2. No Database Migration Strategy

**Severity**: HIGH
**Impact**: Cannot safely deploy schema changes to production

**Current Approach**:
- Using SQLAlchemy `Base.metadata.create_all()` for automatic table creation
- Located in: `src/infrastructure/database/connection.py:34`
- **No migration history tracking**
- **No rollback capability**
- **No schema versioning**

**Tables Created** (October 6, 2025):
1. `users` - User accounts
2. `sessions` - JWT session management
3. `contracts` - Smart contract metadata
4. `scans` - Security scan execution records
5. `vulnerabilities` - Detected security issues

**Required Actions Before Production**:
1. ✅ Install Alembic: `pip install alembic`
2. ✅ Initialize Alembic: `alembic init alembic`
3. ✅ Generate initial migration capturing current schema
4. ✅ Test migration/rollback on staging environment
5. ✅ Document migration process in README

**Migration Commands** (for future reference):
```bash
# Generate new migration after model changes
alembic revision --autogenerate -m "Add new column to contracts"

# Apply migrations
alembic upgrade head

# Rollback one migration
alembic downgrade -1

# View current version
alembic current
```

---

### 3. Pydantic Field Name Conflict (FIXED)

**Severity**: MEDIUM (resolved)
**Status**: ✅ FIXED in commit 6f3d75c

**Issue**:
- Pydantic 2.x validation failed when field name matched imported type
- Error: `"unevaluable-type-annotation"` in `statistics.py`

**Fix Applied**:
```python
# BEFORE (broken):
from datetime import date
class ScanHistoryItem(BaseModel):
    date: date = Field(...)  # ❌ Conflict!

# AFTER (fixed):
from datetime import date as date_type
class ScanHistoryItem(BaseModel):
    date: date_type = Field(...)  # ✅ Works!
```

**File**: `src/presentation/schemas/statistics.py:3`

**Lesson Learned**: Always alias datetime imports when using them as field names in Pydantic models

---

## 📋 NEW FEATURES DEPLOYED

### Database Schema Additions

**New Tables**:

#### `contracts` Table
```sql
CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    address VARCHAR(42) NOT NULL,  -- Ethereum address
    network VARCHAR(50) NOT NULL DEFAULT 'ethereum',
    source_code TEXT,
    bytecode TEXT,
    lines_of_code INTEGER NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, scanning, scanned, failed
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_contracts_user_id ON contracts(user_id);
CREATE INDEX idx_contracts_address ON contracts(address);
```

#### `scans` Table
```sql
CREATE TABLE scans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    scan_type VARCHAR(50) NOT NULL DEFAULT 'full',
    status VARCHAR(20) NOT NULL DEFAULT 'queued',  -- queued, running, completed, failed
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    error_message TEXT,
    critical_count INTEGER NOT NULL DEFAULT 0,
    high_count INTEGER NOT NULL DEFAULT 0,
    medium_count INTEGER NOT NULL DEFAULT 0,
    low_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_scans_contract_id ON scans(contract_id);
CREATE INDEX idx_scans_user_id ON scans(user_id);
```

#### `vulnerabilities` Table
```sql
CREATE TABLE vulnerabilities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_id UUID NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
    contract_id UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    severity VARCHAR(20) NOT NULL,  -- critical, high, medium, low
    status VARCHAR(20) NOT NULL DEFAULT 'open',  -- open, acknowledged, fixed, false_positive
    swc_id VARCHAR(20),  -- Smart Contract Weakness Classification ID
    line_number INTEGER,
    code_snippet TEXT,
    recommendation TEXT,
    detected_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_vulnerabilities_scan_id ON vulnerabilities(scan_id);
CREATE INDEX idx_vulnerabilities_contract_id ON vulnerabilities(contract_id);
CREATE INDEX idx_vulnerabilities_severity ON vulnerabilities(severity);
```

### API Endpoints Added

**Total Endpoints**: 20 (as of October 6, 2025)

#### Contracts Management
- `GET /api/v1/contracts` - List user's contracts (paginated)
- `POST /api/v1/contracts` - Create new contract
- `GET /api/v1/contracts/{contract_id}` - Get contract details
- `PUT /api/v1/contracts/{contract_id}` - Update contract
- `DELETE /api/v1/contracts/{contract_id}` - Delete contract

**Key Features**:
- Pagination (skip/limit query params)
- Automatic line-of-code calculation from source
- Ethereum address validation (0x + 40 hex chars)

#### Vulnerabilities
- `GET /api/v1/vulnerabilities` - List all vulnerabilities (with filters)
- `GET /api/v1/vulnerabilities/{vuln_id}` - Get vulnerability details
- `PATCH /api/v1/vulnerabilities/{vuln_id}/status` - Update status
- `GET /api/v1/vulnerabilities/contracts/{contract_id}/vulnerabilities` - Get by contract

**Filters Available**:
- `severity`: critical, high, medium, low
- `status`: open, acknowledged, fixed, false_positive
- `skip`, `limit`: Pagination

#### Scans
- `GET /api/v1/scans` - List all scans
- `POST /api/v1/scans` - Create new scan
- `GET /api/v1/scans/{scan_id}` - Get scan details
- `GET /api/v1/scans/contracts/{contract_id}/scans` - Get scans for contract

**Automatic Behavior**:
- Contract status automatically set to "scanning" when scan is created
- Scan counts (critical/high/medium/low) stored for quick dashboard queries

#### Statistics & Analytics
- `GET /api/v1/statistics/dashboard` - Aggregated dashboard statistics
- `GET /api/v1/statistics/scan-history` - Historical scan data (30 days)

**Dashboard Stats Calculated**:
```json
{
  "total_scans": 150,
  "total_vulnerabilities": 45,
  "critical_vulnerabilities": 2,
  "high_vulnerabilities": 8,
  "medium_vulnerabilities": 20,
  "low_vulnerabilities": 15,
  "contracts_scanned": 30,
  "average_risk_score": 42.5  // Calculated: (crit*10 + high*5 + med*3 + low*1) / total_scans
}
```

#### File Upload
- `POST /api/v1/upload` - Upload .sol contract files

**Validation**:
- File extension must be `.sol`
- Max file size: 10MB
- UTF-8 encoding required
- Saves to: `/tmp/solidity-contracts/{unique_filename}.sol`

#### User Management
- `GET /api/v1/users/me` - Get current user profile
- `PUT /api/v1/users/me` - Update user profile

#### Health & Info (unchanged)
- `GET /api/v1/health/live` - Liveness probe
- `GET /api/v1/health/ready` - Readiness probe
- `GET /api/v1/health/startup` - Startup probe
- `GET /api/v1/info` - Service info
- `GET /` - Root endpoint

### Authentication

**All endpoints require authentication** except:
- Health checks (`/api/v1/health/*`)
- Service info (`/`, `/api/v1/info`)
- Registration/Login (`/api/v1/auth/*`)

**Authentication Method**: JWT Bearer tokens in `Authorization` header

```bash
# Example authenticated request
curl -H "Authorization: Bearer <access_token>" \
     http://localhost:8001/api/v1/contracts
```

**Security Dependency**: `src/infrastructure/security/dependencies.py:get_current_user()`

---

## 🔧 DEPLOYMENT PROCESS

### Docker Build Requirements

**CRITICAL**: Files must be committed to git before building Docker image

**Why**: The Dockerfile's `COPY . .` command only includes tracked files. Untracked files are silently excluded, leading to import errors at runtime.

**Incorrect Process** (will fail):
```bash
# Create new files
vim src/presentation/schemas/new_schema.py

# Build immediately ❌
docker build -t api-service:latest .

# Result: File not in image, ImportError at runtime
```

**Correct Process**:
```bash
# Create new files
vim src/presentation/schemas/new_schema.py

# Commit first ✅
git add src/presentation/schemas/new_schema.py
git commit -m "Add new schema"

# Now build
docker build -t api-service:latest .
```

### Minikube Build Commands

**For local development with Minikube**:

```bash
# Set Docker environment to use Minikube's Docker daemon
bash -c 'eval $(minikube docker-env) && docker build -t localhost:8080/library/api-service:latest .'

# Verify files are in image
bash -c 'eval $(minikube docker-env) && docker run --rm localhost:8080/library/api-service:latest ls -la /app/src/presentation/schemas/'

# Deploy updated image
kubectl delete pods -n api-service-local -l app=api-service

# Or force rollout
kubectl set image deployment/api-service api-service=localhost:8080/library/api-service:latest -n api-service-local
kubectl rollout status deployment/api-service -n api-service-local
```

### Environment-Specific Configurations

**The Dockerfile is environment-agnostic**. Configuration is managed through:

1. **Kubernetes ConfigMaps** (`k8s/overlays/{env}/configmap.yaml`)
2. **Kubernetes Secrets** (`k8s/overlays/{env}/secrets.yaml`)
3. **Environment Variables** (injected at runtime)

**Key Variables**:
```yaml
# Required for all environments
DATABASE_URL: postgresql+asyncpg://user:pass@host:5432/dbname
REDIS_URL: redis://host:6379/0
JWT_SECRET_KEY: <random-256-bit-key>
SESSION_SECRET: <random-256-bit-key>

# Optional
ENVIRONMENT: local|staging|production
LOG_LEVEL: DEBUG|INFO|WARNING|ERROR
ENABLE_DEBUG: true|false
CORS_ORIGINS: ["http://localhost:3000", "https://app.example.com"]
```

**Deployment Overlays**:
- `k8s/overlays/local/` - Minikube development
- `k8s/overlays/staging/` - Staging environment (to be created)
- `k8s/overlays/production/` - Production environment (to be created)

---

## 🧪 TESTING STATUS

### Manual Testing Completed ✅

**Infrastructure**:
- ✅ Docker image builds successfully (396MB)
- ✅ Kubernetes deployment rolls out (5 replicas)
- ✅ All 5 database tables created automatically
- ✅ Port forwarding works (8001:8000)

**Endpoints**:
- ✅ Root endpoint (`/`) returns service info
- ✅ Service info endpoint (`/api/v1/info`) returns all paths
- ✅ OpenAPI spec available at `/openapi.json`
- ✅ Interactive docs available at `/docs`

**Known Failures** ❌:
- ❌ User registration (bcrypt error)
- ❌ User login (bcrypt error)
- ❌ All protected endpoints (cannot get auth token)

### Testing Blockers

**Cannot test** (blocked by authentication bug):
- Contract creation
- Vulnerability listing
- Scan execution
- Statistics aggregation
- File upload
- User profile management

**Workaround Attempted**:
- Manually inserted user into database with pre-hashed password
- Login still failed due to bcrypt verification error

### Automated Testing TODO

**Unit Tests** (none currently exist):
```bash
# Required test coverage before production:
tests/
  unit/
    test_password_hashing.py       # ← CRITICAL
    test_jwt_tokens.py
    test_models.py
  integration/
    test_auth_flow.py
    test_contract_crud.py
    test_vulnerability_queries.py
    test_statistics_aggregation.py
  e2e/
    test_full_scan_workflow.py
```

**Test Framework**: pytest + pytest-asyncio
**Target Coverage**: 80% minimum

---

## 📊 PERFORMANCE CONSIDERATIONS

### Database Queries

**Potential N+1 Query Issues**:

1. **Contracts List with Vulnerability Counts**:
   - Location: `src/presentation/api/v1/endpoints/contracts.py:45-60`
   - Current: Loops through contracts to count vulnerabilities
   - **TODO**: Add SQL aggregation to fetch counts in single query

```python
# Current (N+1):
for contract in contracts:
    vuln_count = await db.execute(
        select(func.count()).where(VulnerabilityModel.contract_id == contract.id)
    )

# Recommended:
stmt = (
    select(
        ContractModel,
        func.count(VulnerabilityModel.id).label('vuln_count')
    )
    .outerjoin(VulnerabilityModel)
    .group_by(ContractModel.id)
)
```

2. **Statistics Dashboard Aggregations**:
   - Location: `src/presentation/api/v1/endpoints/statistics.py:25-80`
   - Current: Multiple separate COUNT queries
   - **TODO**: Combine into single query with CASE statements

### Pagination Limits

**Default**: 100 items per page
**Maximum**: Should add upper limit (e.g., 1000) to prevent resource exhaustion

```python
# Add to all list endpoints:
def validate_limit(limit: int = Query(100, le=1000)):
    return limit
```

### Caching Opportunities

**Statistics Dashboard** (`/api/v1/statistics/dashboard`):
- Data changes infrequently (only after scans complete)
- **Recommendation**: Cache for 5 minutes using Redis
- Implementation: `@cache(expire=300)` decorator

**Scan History** (`/api/v1/statistics/scan-history`):
- Historical data doesn't change
- **Recommendation**: Cache for 1 hour

---

## 🔐 SECURITY CONSIDERATIONS

### Current Implementation

✅ **Good**:
- JWT token-based authentication
- Password hashing with bcrypt (when working)
- UUID primary keys (non-sequential)
- CORS configuration
- Parameterized SQL queries (SQLAlchemy prevents injection)

⚠️ **Needs Improvement**:
1. **No rate limiting** - DDoS vulnerable
2. **No request size limits** - Can upload huge files
3. **No input sanitization** - XSS risk in stored data
4. **Sessions never expire** - 7-day refresh tokens stored indefinitely
5. **No HTTPS enforcement** - Tokens sent over plain HTTP in local
6. **No CSRF protection** - Vulnerable if using cookies

### Security Enhancements Required

See: `/Users/pwner/Git/ABS/docs/Sprints/Sprint-1/Task-1.18-Security-Hardening.md`

**Priority P0** (before production):
- [ ] HttpOnly cookies for token storage
- [ ] Token rotation on refresh
- [ ] HTTPS only in production
- [ ] Secrets management (Vault/AWS Secrets Manager)
- [ ] Kubernetes NetworkPolicies
- [ ] Database connection encryption (TLS)

**Priority P1** (within 2 weeks of launch):
- [ ] Input validation framework
- [ ] Rate limiting (per-IP and per-user)
- [ ] Tightened CORS policy
- [ ] Web Application Firewall (WAF)

---

## 🚀 PRE-PRODUCTION CHECKLIST

### Code Quality
- [ ] Fix bcrypt password hashing bug
- [ ] Add Alembic database migrations
- [ ] Write unit tests (80% coverage target)
- [ ] Write integration tests for all endpoints
- [ ] Add request/response logging
- [ ] Implement structured logging (JSON format)
- [ ] Add OpenTelemetry tracing

### Security
- [ ] Complete Task 1.18 security hardening (71 hours estimated)
- [ ] Penetration testing
- [ ] Dependency vulnerability scan (`safety check`)
- [ ] SAST scanning (Bandit, Semgrep)
- [ ] Secrets rotation plan documented

### Infrastructure
- [ ] Set up staging environment matching production
- [ ] Configure database backups (daily + PITR)
- [ ] Set up monitoring alerts (Prometheus/Grafana)
- [ ] Configure log aggregation (ELK/Loki)
- [ ] Load testing (expected: 100 RPS)
- [ ] Disaster recovery plan documented

### Documentation
- [ ] API documentation (OpenAPI complete ✅, add examples)
- [ ] Deployment runbook
- [ ] Incident response playbook
- [ ] Database schema documentation
- [ ] Architecture diagrams (C4 model)

### Compliance
- [ ] GDPR compliance review (user data handling)
- [ ] Data retention policy defined
- [ ] Privacy policy updated
- [ ] Terms of service updated
- [ ] Security audit completed

---

## 📝 ENVIRONMENT VARIABLES REFERENCE

### Required Variables

```bash
# Database
DATABASE_URL="postgresql+asyncpg://user:password@host:5432/database"

# Redis (Session Storage)
REDIS_URL="redis://host:6379/0"

# JWT Authentication
JWT_SECRET_KEY="<256-bit-random-string>"  # openssl rand -hex 32
JWT_ALGORITHM="HS256"
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# Session Management
SESSION_SECRET="<256-bit-random-string>"  # openssl rand -hex 32

# Application
ENVIRONMENT="local|staging|production"
LOG_LEVEL="DEBUG|INFO|WARNING|ERROR"
DEBUG="true|false"

# CORS
CORS_ORIGINS='["http://localhost:3000"]'  # JSON array

# File Upload
UPLOAD_DIR="/tmp/solidity-contracts"
MAX_UPLOAD_SIZE=10485760  # 10MB in bytes
```

### Optional Variables

```bash
# Metrics & Monitoring
ENABLE_METRICS="true"
METRICS_PORT=9090
PROMETHEUS_URL="http://prometheus:9090"

# External Services (future)
ETHERSCAN_API_KEY=""
INFURA_PROJECT_ID=""
ANALYSIS_SERVICE_URL="http://analysis-engine:8080"
```

---

## 🆘 TROUBLESHOOTING

### Common Issues

#### 1. "Import Error: No module named 'src.presentation.schemas.contracts'"

**Cause**: File not committed before Docker build
**Solution**:
```bash
git add src/presentation/schemas/contracts.py
git commit -m "Add contracts schema"
# Rebuild image
```

#### 2. Pods stuck in CrashLoopBackOff

**Cause**: Application startup failure (usually import errors)
**Diagnosis**:
```bash
kubectl logs -n api-service-local -l app=api-service --tail=50
kubectl describe pod -n api-service-local <pod-name>
```

#### 3. Database connection refused

**Check**:
```bash
# Verify PostgreSQL is running
kubectl get pods -n postgresql-local

# Test connection from pod
kubectl exec -it -n api-service-local <pod-name> -- \
  python3 -c "import asyncpg; asyncpg.connect('postgresql://...')"
```

#### 4. "password cannot be longer than 72 bytes"

**Status**: Known bug, under investigation
**Workaround**: Manually insert users into database with pre-hashed passwords
**Priority**: CRITICAL - blocks all authentication

---

## 📞 SUPPORT CONTACTS

**For deployment issues**:
- DevOps Lead: [Contact Info]
- Backend Team: [Contact Info]

**For security concerns**:
- Security Team: [Contact Info]
- On-call rotation: [PagerDuty/OpsGenie]

**Escalation path**:
1. Check this document
2. Search GitHub issues
3. Post in #engineering Slack channel
4. Page on-call engineer (production only)

---

## 📚 RELATED DOCUMENTATION

- **Architecture**: `/Users/pwner/Git/ABS/docs/architecture/`
- **Security Plan**: `/Users/pwner/Git/ABS/docs/Sprints/Sprint-1/Task-1.18-Security-Hardening.md`
- **Sprint Plan**: `/Users/pwner/Git/ABS/docs/sprint-plan_new.md`
- **API Docs**: `http://localhost:8001/docs` (when running)
- **Database Schema**: See "Database Schema Additions" section above

---

## 🔄 CHANGE LOG

| Date | Author | Changes |
|------|--------|---------|
| 2025-10-06 | Claude Code | Initial deployment notes created |
| 2025-10-06 | Claude Code | Documented bcrypt bug and 20 new endpoints |
| 2025-10-06 | Claude Code | Added pre-production checklist and troubleshooting |

---

**Next Review Date**: Before staging deployment
**Document Owner**: Backend Team Lead
