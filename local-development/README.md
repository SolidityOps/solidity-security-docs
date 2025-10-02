# Local Development Environment Documentation

This directory contains documentation specific to the local development environment setup for the SolidityOps platform. **These configurations and workarounds are for local development only and should NOT be deployed to production.**

## 📁 Documentation Structure

- [`setup-summary.md`](./setup-summary.md) - Complete overview of what was built and configured
- [`shared-library-build.md`](./shared-library-build.md) - Shared library build process and workarounds
- [`infrastructure-fixes.md`](./infrastructure-fixes.md) - Database and service fixes applied
- [`docker-modifications.md`](./docker-modifications.md) - Docker image build modifications
- [`deployment-verification.md`](./deployment-verification.md) - How to verify the local environment
- [`production-differences.md`](./production-differences.md) - Key differences from production setup

## ⚠️ **CRITICAL WARNING**

**All files in this directory are LOCAL DEVELOPMENT ONLY**

The configurations, workarounds, and modifications documented here are specifically for local minikube development and contain:

- Temporary fixes for dependency issues
- Local-only database configurations
- Development-mode security settings
- Simplified build processes
- Local registry configurations

**DO NOT use these configurations in production environments.**

## 🎯 Purpose

This documentation serves to:

1. **Document the local development setup** for future reference
2. **Prevent production deployment** of development-only configurations
3. **Enable team onboarding** with proper local environment setup
4. **Track modifications made** for local development compatibility
5. **Maintain separation** between local and production configurations

## 🚀 Quick Start

For setting up the local development environment, follow the documents in this order:

1. Read [`setup-summary.md`](./setup-summary.md) for an overview
2. Follow [`shared-library-build.md`](./shared-library-build.md) for dependency setup
3. Review [`infrastructure-fixes.md`](./infrastructure-fixes.md) for service configurations
4. Use [`deployment-verification.md`](./deployment-verification.md) to verify setup

## 📋 Environment Status

The local development environment includes:

- ✅ minikube cluster (7GB RAM, 6 CPUs, 80GB disk)
- ✅ PostgreSQL (official postgres:15 image)
- ✅ Redis (official redis:7 image)
- ✅ Vault (development mode)
- ✅ Monitoring stack (Prometheus + Grafana)
- ✅ Local Docker registry (localhost:5000)
- ✅ Shared library (pure Python build)
- ✅ API service image (built and tested)

Last Updated: October 2, 2025
Environment Version: Development v1.0