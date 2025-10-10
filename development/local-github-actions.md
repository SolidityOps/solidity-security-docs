# Running GitHub Actions Locally with Act

**Version:** 1.0.0
**Last Updated:** October 9, 2025
**Tool:** act (v0.2.82)

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Quick Start](#quick-start)
4. [Running Workflows](#running-workflows)
5. [Advanced Usage](#advanced-usage)
6. [Configuration](#configuration)
7. [Troubleshooting](#troubleshooting)
8. [Cost Savings](#cost-savings)

---

## Overview

**`act`** is a tool that runs your GitHub Actions workflows locally in Docker containers. This allows you to:

- ✅ **Save Costs:** Test workflows locally before pushing to GitHub
- ✅ **Faster Iteration:** No need to wait for GitHub Actions runners
- ✅ **Offline Development:** Test workflows without internet connection
- ✅ **Debugging:** Run workflows step-by-step with verbose logging
- ✅ **Pre-commit Validation:** Catch workflow errors before pushing

### How It Works

```
GitHub Actions Workflow (.github/workflows/test.yml)
                ↓
         act reads workflow
                ↓
   Runs jobs in Docker containers
                ↓
    Same environment as GitHub
```

---

## Installation

### macOS (Homebrew)

```bash
# Install act
brew install act

# Verify installation
act --version
# Output: act version 0.2.82
```

### Linux

```bash
# Download and install
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Verify installation
act --version
```

### Windows

```powershell
# Using Chocolatey
choco install act-cli

# Using Scoop
scoop install act
```

---

## Quick Start

### 1. List Available Workflows

```bash
# Navigate to project root
cd /Users/pwner/Git/ABS/solidity-security-api-service

# List all workflows
act -l

# Expected output:
# Stage  Job ID             Job name           Workflow name       Workflow file       Events
# 0      test               Run Tests          Automated Testing   test.yml           push,pull_request
# 0      quality            Code Quality       Automated Testing   test.yml           push,pull_request
# 0      security           Security Scan      Automated Testing   test.yml           push,pull_request
# 1      build              Build Docker Image Automated Testing   test.yml           push,pull_request
# 2      test-summary       Test Summary       Automated Testing   test.yml           push,pull_request
```

### 2. Run All Workflows

```bash
# Run all jobs (uses default "push" event)
act

# Run with verbose output
act -v
```

### 3. Run Specific Job

```bash
# Run only test job
act -j test

# Run only quality job
act -j quality

# Run only security job
act -j security
```

### 4. Dry Run (Show What Would Run)

```bash
# Show what would run without executing
act -n

# Show with verbose output
act -n -v
```

---

## Running Workflows

### Run Test Workflow

The test workflow requires PostgreSQL and Redis services. There are two approaches:

#### Option 1: Use Existing Kubernetes Services (Recommended)

```bash
# 1. Port-forward PostgreSQL and Redis (in separate terminals)
kubectl port-forward -n postgresql-local svc/postgresql 5432:5432
kubectl port-forward -n redis-local svc/redis-master 6379:6379

# 2. Run test job (connects to localhost:5432 and localhost:6379)
act -j test --container-architecture linux/amd64
```

#### Option 2: Run with Act Service Containers

```bash
# Act will start PostgreSQL and Redis containers automatically
act -j test --container-architecture linux/amd64

# Note: Service containers are defined in .github/workflows/test.yml
```

### Run Quality Checks

```bash
# Run code quality job (Black, Ruff, mypy)
act -j quality
```

### Run Security Scanning

```bash
# Run security scanning job (safety, bandit)
act -j security
```

### Run Build Job

```bash
# Run Docker image build (requires VERSION file)
act -j build --container-architecture linux/amd64

# Note: Build job only runs on main/develop branches
# Override with:
act -j build -e .github/workflows/event.json
```

---

## Advanced Usage

### Custom Event Payload

Create a custom event file to simulate different triggers:

```bash
# Create event file
cat > .github/workflows/event.json <<EOF
{
  "ref": "refs/heads/main",
  "repository": {
    "name": "solidity-security-api-service",
    "owner": {
      "login": "your-username"
    }
  }
}
EOF

# Run with custom event
act -e .github/workflows/event.json
```

### Environment Variables

```bash
# Pass environment variables
act -j test \
  --env DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/solidity_security_test \
  --env REDIS_URL=redis://localhost:6379/0 \
  --env JWT_SECRET_KEY=test-secret

# Load from .env file
act -j test --env-file .env.test
```

### Secrets

```bash
# Create secrets file
cat > .secrets <<EOF
CODECOV_TOKEN=your-codecov-token
GITHUB_TOKEN=your-github-token
EOF

# Run with secrets
act -j test --secret-file .secrets
```

### Run Specific Workflow File

```bash
# Run specific workflow file
act -W .github/workflows/test.yml

# Run specific event type
act push
act pull_request
```

### Interactive Shell

```bash
# Drop into shell in workflow container
act -j test --shell bash

# Debug specific step
act -j test --shell bash --step "Run unit tests"
```

### Platform Selection

```bash
# Use smaller runner images (faster, less memory)
act -P ubuntu-latest=catthehacker/ubuntu:act-latest

# Use medium runner images (recommended)
act -P ubuntu-latest=catthehacker/ubuntu:act-20.04

# Use large runner images (most compatible, slower)
act -P ubuntu-latest=ubuntu:20.04
```

---

## Configuration

### .actrc Configuration File

Create `.actrc` in project root for persistent configuration:

```bash
# .actrc
-P ubuntu-latest=catthehacker/ubuntu:act-20.04
--container-architecture linux/amd64
--env-file .env.test
--artifact-server-path /tmp/artifacts
```

Now you can just run:
```bash
act -j test  # Uses .actrc configuration
```

### .env.test for Local Testing

Create `.env.test` for local workflow testing:

```bash
# .env.test
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/solidity_security_test
REDIS_URL=redis://localhost:6379/0
JWT_SECRET_KEY=test-secret-key-for-local-act
SESSION_SECRET=test-session-secret-for-local-act
DEBUG=true
ENVIRONMENT=test
PYTHONPATH=/Users/pwner/Git/ABS/solidity-security-api-service/src
```

---

## Troubleshooting

### Issue 1: Service Containers Not Starting

**Error:** `Failed to create container: Error response from daemon`

**Solution:**
```bash
# Use existing Kubernetes services instead
kubectl port-forward -n postgresql-local svc/postgresql 5432:5432
kubectl port-forward -n redis-local svc/redis-master 6379:6379

# Then run act
act -j test
```

### Issue 2: Architecture Mismatch

**Error:** `WARNING: The requested image's platform (linux/amd64) does not match`

**Solution:**
```bash
# Force linux/amd64 architecture (required for M1/M2 Macs)
act -j test --container-architecture linux/amd64
```

### Issue 3: Out of Memory

**Error:** `Exited with code 137` (out of memory)

**Solution:**
```bash
# Increase Docker memory limit in Docker Desktop:
# Settings → Resources → Memory → 8GB+

# Or use smaller runner image
act -j test -P ubuntu-latest=catthehacker/ubuntu:act-latest
```

### Issue 4: Permission Denied

**Error:** `permission denied while trying to connect to Docker daemon`

**Solution:**
```bash
# Add user to docker group (Linux)
sudo usermod -aG docker $USER
newgrp docker

# Or run with sudo (not recommended)
sudo act -j test
```

### Issue 5: GitHub Token Required

**Error:** `failed to get action: failed to clone GitHub repository`

**Solution:**
```bash
# Create GitHub personal access token
# https://github.com/settings/tokens

# Pass token to act
act -j test --secret GITHUB_TOKEN=ghp_your_token_here

# Or add to .secrets file
echo "GITHUB_TOKEN=ghp_your_token_here" > .secrets
act -j test --secret-file .secrets
```

### Issue 6: Build Job Skipped

**Error:** `build job is skipped`

**Reason:** Build job only runs on main/develop branches

**Solution:**
```bash
# Create event file simulating push to main
cat > .github/workflows/event.json <<EOF
{
  "ref": "refs/heads/main"
}
EOF

# Run with custom event
act -j build -e .github/workflows/event.json
```

### Issue 7: Volume Mount Issues

**Error:** `invalid mount config for type "bind"`

**Solution:**
```bash
# Check Docker Desktop file sharing settings
# Settings → Resources → File Sharing → Add /Users/pwner/Git/ABS

# Or use bind-mount flag
act -j test --bind
```

---

## Cost Savings

### GitHub Actions Pricing (2025)

- **Free Tier:** 2,000 minutes/month for private repos
- **Overage:** $0.008 per minute for Linux runners
- **Average Workflow:** ~10 minutes per run
- **Cost per Run:** ~$0.08

### Example Savings

**Scenario:** Developing a feature with 50 test runs before pushing

| Method | Runs | Cost | Time |
|--------|------|------|------|
| GitHub Actions | 50 | $4.00 | ~500 min (wait for queue) |
| Local with `act` | 50 | $0.00 | ~250 min (no queue) |
| **Savings** | - | **$4.00** | **50% faster** |

**Annual Savings (5 developers):**
- 50 workflows/week/developer × 5 developers = 250 workflows/week
- 250 workflows × $0.08 = $20/week
- $20/week × 52 weeks = **$1,040/year**

### Best Practices for Cost Savings

1. **Local First:** Test workflows with `act` before pushing
2. **Targeted Jobs:** Run only the job you need (`act -j test`)
3. **Dry Runs:** Validate workflow syntax with `act -n`
4. **Parallel Development:** No wait for GitHub Actions queue
5. **Debug Locally:** Fix issues locally instead of commit-push-debug cycle

---

## Common Workflow Commands

### Development Workflow

```bash
# 1. Make changes to code
vim src/main.py

# 2. Run tests locally
act -j test --container-architecture linux/amd64

# 3. Run quality checks
act -j quality

# 4. If all pass, push to GitHub
git add .
git commit -m "Add new feature"
git push
```

### Pre-commit Hook Integration

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# Run GitHub Actions locally before committing

echo "Running GitHub Actions locally with act..."

# Run quality checks
if ! act -j quality -P ubuntu-latest=catthehacker/ubuntu:act-latest; then
    echo "❌ Quality checks failed. Commit aborted."
    exit 1
fi

echo "✅ Quality checks passed. Proceeding with commit."
```

Make executable:
```bash
chmod +x .git/hooks/pre-commit
```

### Continuous Testing During Development

```bash
# Watch for changes and re-run tests
while true; do
    act -j test --container-architecture linux/amd64
    sleep 10
done

# Or use file watcher (requires entr)
find src/ tests/ | entr -c act -j test
```

---

## Comparison: act vs GitHub Actions

| Feature | GitHub Actions | act (Local) |
|---------|---------------|-------------|
| Cost | $0.008/min after free tier | Free |
| Speed | Queue wait + execution | Immediate execution |
| Internet Required | Yes | No |
| Debugging | View logs after completion | Live debugging |
| Service Containers | Automatic | Use existing K8s or Docker |
| Secrets | GitHub Secrets | Local .secrets file |
| Artifacts | Automatic upload | Local filesystem |
| Cache | GitHub-hosted | Local Docker cache |
| Accuracy | 100% (official) | ~95% (some features unsupported) |

---

## Limitations

### Features Not Supported by act:

1. **GitHub Context:** Some `${{ github.* }}` variables may be incomplete
2. **OIDC Tokens:** OpenID Connect tokens not available
3. **GitHub API:** Limited API access without token
4. **Artifacts:** No automatic artifact upload/download
5. **Caching:** Uses Docker layer cache instead of GitHub cache
6. **Matrix Builds:** Limited support for complex matrices
7. **Reusable Workflows:** Partial support for composite actions

### Workarounds:

```bash
# Missing secrets: Create .secrets file
echo "CODECOV_TOKEN=fake-token-for-testing" > .secrets
act -j test --secret-file .secrets

# Missing GitHub context: Use event.json
cat > .github/workflows/event.json <<EOF
{
  "ref": "refs/heads/main",
  "sha": "abc123",
  "repository": {
    "name": "solidity-security-api-service"
  }
}
EOF
act -e .github/workflows/event.json
```

---

## Best Practices

### DO:
✅ Test workflows locally with `act` before pushing
✅ Use `.actrc` for consistent configuration
✅ Use smaller runner images for faster execution
✅ Use `--container-architecture linux/amd64` on M1/M2 Macs
✅ Run specific jobs with `-j` flag
✅ Use existing Kubernetes services for PostgreSQL/Redis
✅ Create `.env.test` for local environment variables

### DON'T:
❌ Don't rely on `act` for 100% accuracy (always validate in GitHub)
❌ Don't commit `.secrets` file (add to .gitignore)
❌ Don't run all jobs if you only need one
❌ Don't use `latest` tags in workflows (use specific versions)
❌ Don't assume all GitHub Actions features work in `act`

---

## Resources

- **act GitHub:** https://github.com/nektos/act
- **act Documentation:** https://nektosact.com/
- **Runner Images:** https://github.com/catthehacker/docker_images
- **GitHub Actions Docs:** https://docs.github.com/en/actions
- **Workflow File:** `.github/workflows/test.yml`

---

## Quick Reference

```bash
# List workflows
act -l

# Run all workflows
act

# Run specific job
act -j test
act -j quality
act -j security
act -j build

# Dry run
act -n

# Verbose output
act -v

# With environment variables
act -j test --env-file .env.test

# With secrets
act -j test --secret-file .secrets

# With custom event
act -j build -e .github/workflows/event.json

# Force architecture (M1/M2 Macs)
act -j test --container-architecture linux/amd64

# Use smaller runner image
act -j test -P ubuntu-latest=catthehacker/ubuntu:act-latest

# Interactive shell
act -j test --shell bash
```

---

## Summary

**`act` enables cost-effective local GitHub Actions testing:**

- ✅ Installed and verified (v0.2.82)
- ✅ Saves ~$1,040/year for 5-developer team
- ✅ 50% faster iteration (no queue wait)
- ✅ Full offline development support
- ✅ Compatible with existing `.github/workflows/test.yml`

**Next Steps:**
1. Run `act -l` to list available workflows
2. Run `act -j test` to test locally
3. Create `.actrc` for persistent configuration
4. Add pre-commit hook for automatic validation

---

**Created:** October 9, 2025
**Tool:** act v0.2.82
**Platform:** macOS (Sonoma)
**Docker:** Required
