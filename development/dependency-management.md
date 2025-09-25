# Dependency Management Guide

## Overview

This guide covers dependency management strategies across all services in the Solidity Security Platform. Each service category uses language-specific dependency management tools while maintaining consistency and shared library integration.

## Platform Architecture

### Service Dependencies Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                 Shared Library Foundation                       │
│  solidity-security-shared (Rust + Python PyO3 + TypeScript)   │
└─────────────────────────────────────────────────────────────────┘
                                  │
    ┌─────────────────────────────┼─────────────────────────────┐
    │                             │                             │
┌───▼────┐                   ┌────▼───┐                   ┌─────▼──┐
│ Python │                   │TypeScript│                │  Rust  │
│Services│                   │Services  │                │Service │
│(6 svc) │                   │(4 svc)   │                │(1 svc) │
└────────┘                   └──────────┘                └────────┘
```

### Dependency Categories

1. **Production Dependencies**: Core functionality and runtime requirements
2. **Development Dependencies**: Build tools, testing, linting, formatting
3. **Shared Dependencies**: Multi-language shared library components
4. **Optional Dependencies**: Feature flags and environment-specific needs

## Python Dependency Management

### Structure and Organization

#### Requirements Files Structure
```
requirements/
├── base.txt        # Production dependencies
├── dev.txt         # Development dependencies (includes base.txt)
├── test.txt        # Testing dependencies (includes dev.txt)
└── optional.txt    # Optional/feature-specific dependencies
```

#### Base Production Dependencies (base.txt)

**Core Framework Dependencies:**
```bash
# Web framework and async support
fastapi>=0.104.0,<1.0.0
uvicorn[standard]>=0.24.0,<1.0.0
pydantic>=2.4.0,<3.0.0
pydantic-settings>=2.0.0,<3.0.0

# Database and ORM
sqlalchemy>=2.0.0,<3.0.0
alembic>=1.12.0,<2.0.0
asyncpg>=0.29.0,<1.0.0  # PostgreSQL async driver

# Redis and caching
redis>=5.0.0,<6.0.0
aioredis>=2.0.0,<3.0.0

# HTTP client
httpx>=0.25.0,<1.0.0
aiohttp>=3.9.0,<4.0.0

# Task queue (Celery services)
celery[redis]>=5.3.0,<6.0.0

# Shared library integration
-e ../solidity-security-shared/python
```

**Service-Specific Dependencies:**

*Intelligence Engine (ML/AI):*
```bash
# Machine learning libraries
tensorflow>=2.13.0,<3.0.0
scikit-learn>=1.3.0,<2.0.0
numpy>=1.24.0,<2.0.0
pandas>=2.1.0,<3.0.0
torch>=2.1.0,<3.0.0
transformers>=4.35.0,<5.0.0
```

*Tool Integration:*
```bash
# Process and system management
psutil>=5.9.0,<6.0.0
docker>=6.1.0,<7.0.0
kubernetes>=28.1.0,<29.0.0
```

*Notification Service:*
```bash
# WebSocket and real-time
websockets>=11.0.0,<12.0.0
python-socketio>=5.9.0,<6.0.0

# Email and messaging
aiosmtplib>=3.0.0,<4.0.0
jinja2>=3.1.0,<4.0.0
```

#### Development Dependencies (dev.txt)
```bash
# Include production dependencies
-r base.txt

# Testing framework
pytest>=7.4.0,<8.0.0
pytest-asyncio>=0.21.0,<1.0.0
pytest-cov>=4.1.0,<5.0.0
pytest-mock>=3.12.0,<4.0.0
httpx>=0.25.0  # For testing FastAPI

# Code formatting and linting
black>=23.9.0,<24.0.0
isort>=5.12.0,<6.0.0
ruff>=0.1.0,<1.0.0

# Type checking
mypy>=1.6.0,<2.0.0
types-redis>=4.6.0
types-requests>=2.31.0

# Development tools
ipython>=8.16.0,<9.0.0
watchdog>=3.0.0,<4.0.0  # File watching for auto-reload
```

#### Testing Dependencies (test.txt)
```bash
# Include development dependencies
-r dev.txt

# Additional testing tools
factory-boy>=3.3.0,<4.0.0
faker>=19.12.0,<20.0.0
pytest-benchmark>=4.0.0,<5.0.0
pytest-xdist>=3.3.0,<4.0.0  # Parallel test execution
coverage[toml]>=7.3.0,<8.0.0
```

### Python Configuration Files

#### pyproject.toml
```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "solidity-security-{service-name}"
dynamic = ["version"]
description = "Solidity Security Platform - {Service Description}"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [
    {name = "Solidity Security Team"}
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

[project.optional-dependencies]
dev = ["pytest", "black", "isort", "ruff", "mypy"]
test = ["pytest-cov", "pytest-asyncio", "factory-boy"]

# Tool configurations
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.venv
  | \.env
  | build
  | dist
)/
'''

[tool.isort]
profile = "black"
multi_line_output = 3
line_length = 88
known_first_party = ["src"]
known_third_party = ["fastapi", "sqlalchemy", "pydantic"]

[tool.ruff]
target-version = "py311"
line-length = 88
select = [
    "E",  # pycodestyle errors
    "W",  # pycodestyle warnings
    "F",  # pyflakes
    "I",  # isort
    "C",  # flake8-comprehensions
    "B",  # flake8-bugbear
]
ignore = [
    "E501",  # line too long, handled by black
]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
disallow_untyped_decorators = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",
    "--tb=short",
    "--strict-config",
    "--strict-markers",
]
markers = [
    "unit: Unit tests",
    "integration: Integration tests",
    "slow: Slow tests",
]
asyncio_mode = "auto"

[tool.coverage.run]
source = ["src"]
omit = [
    "*/tests/*",
    "*/venv/*",
    "*/.venv/*",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
]
```

### Installation and Management

#### Virtual Environment Setup
```bash
# Create isolated environment
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# or
venv\Scripts\activate     # Windows

# Upgrade pip and core tools
pip install --upgrade pip setuptools wheel
```

#### Development Installation
```bash
# Install development dependencies
pip install -r requirements/dev.txt

# Install in editable mode for development
pip install -e .

# Install shared library in development mode
pip install -e ../solidity-security-shared/python
```

#### Dependency Updates
```bash
# Check outdated packages
pip list --outdated

# Update specific package
pip install --upgrade package-name

# Generate new requirements
pip-tools compile requirements.in --upgrade
```

## TypeScript Dependency Management

### Package.json Configuration

#### Core Dependencies Structure

**Base Application Dependencies:**
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.18.0",
    "react-hook-form": "^7.47.0",

    "@tanstack/react-query": "^5.0.0",
    "zustand": "^4.4.0",

    "@headlessui/react": "^1.7.0",
    "@heroicons/react": "^2.0.0",
    "tailwindcss": "^3.3.0",

    "axios": "^1.6.0",
    "clsx": "^2.0.0",
    "date-fns": "^2.30.0",

    "@solidity-security/shared": "file:../solidity-security-shared/typescript"
  }
}
```

**Development Dependencies:**
```json
{
  "devDependencies": {
    "@types/react": "^18.2.37",
    "@types/react-dom": "^18.2.15",
    "@types/node": "^20.8.0",

    "@vitejs/plugin-react": "^4.1.0",
    "vite": "^4.5.0",
    "vitest": "^0.34.0",
    "@vitest/ui": "^0.34.0",

    "typescript": "^5.2.0",
    "@typescript-eslint/eslint-plugin": "^6.9.0",
    "@typescript-eslint/parser": "^6.9.0",
    "eslint": "^8.52.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.4",

    "prettier": "^3.0.0",
    "prettier-plugin-tailwindcss": "^0.5.0",

    "@testing-library/react": "^13.4.0",
    "@testing-library/jest-dom": "^6.1.0",
    "@testing-library/user-event": "^14.5.0"
  }
}
```

#### Service-Specific Dependencies

**UI Core (Component Library):**
```json
{
  "name": "@solidity-security/ui-core",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./styles": "./dist/styles.css"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@rollup/plugin-typescript": "^11.1.0",
    "rollup": "^3.29.0",
    "rollup-plugin-dts": "^6.1.0"
  }
}
```

**Dashboard/Findings/Analysis Apps:**
```json
{
  "dependencies": {
    "@solidity-security/ui-core": "workspace:*",
    "recharts": "^2.8.0",
    "react-table": "^7.8.0",
    "@monaco-editor/react": "^4.6.0"
  }
}
```

#### Scripts Configuration
```json
{
  "scripts": {
    "dev": "vite --port 3000",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:run": "vitest run",
    "coverage": "vitest run --coverage",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext ts,tsx --fix",
    "type-check": "tsc --noEmit",
    "format": "prettier --write \"src/**/*.{ts,tsx,js,jsx}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,js,jsx}\""
  }
}
```

### TypeScript Configuration

#### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/hooks/*": ["./src/hooks/*"],
      "@/utils/*": ["./src/utils/*"],
      "@shared/*": ["../solidity-security-shared/typescript/src/*"]
    }
  },
  "include": ["src", "vite.config.ts"],
  "references": [
    { "path": "./tsconfig.node.json" },
    { "path": "../solidity-security-shared/typescript" }
  ]
}
```

### Package Management

#### Installation Commands
```bash
# Install all dependencies
npm install

# Install specific dependency
npm install package-name

# Install development dependency
npm install --save-dev package-name

# Install shared library
npm install file:../solidity-security-shared/typescript
```

#### Workspace Management (Optional)
```json
{
  "workspaces": [
    "solidity-security-ui-core",
    "solidity-security-dashboard",
    "solidity-security-findings",
    "solidity-security-analysis",
    "solidity-security-shared/typescript"
  ]
}
```

## Rust Dependency Management

### Cargo.toml Configuration

#### Core Dependencies
```toml
[package]
name = "solidity-security-contract-parser"
version = "0.1.0"
edition = "2021"
rust-version = "1.75.0"
description = "High-performance Solidity contract parsing service"
license = "MIT"
repository = "https://github.com/your-org/solidity-security-contract-parser"

[dependencies]
# HTTP server framework
axum = "0.7"
tokio = { version = "1.35", features = ["full"] }
tower = { version = "0.4", features = ["util"] }
tower-http = { version = "0.5", features = ["cors", "trace", "fs"] }

# Serialization and data handling
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.6", features = ["serde", "v4"] }

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Async utilities
futures = "0.3"
async-trait = "0.1"

# Logging and tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Configuration
config = "0.13"
dotenvy = "0.15"

# Shared library integration
solidity-security-shared = { path = "../solidity-security-shared/rust" }

# Parser-specific dependencies
tree-sitter = "0.20"
tree-sitter-solidity = "1.2"

[dev-dependencies]
# Testing
tokio-test = "0.4"
criterion = { version = "0.5", features = ["html_reports"] }
mockall = "0.12"

# Test utilities
serde_test = "1.0"
tempfile = "3.8"

[[bench]]
name = "parsing_benchmark"
harness = false

[features]
default = ["tracing"]
tracing = ["dep:tracing", "dep:tracing-subscriber"]
metrics = []

[profile.dev]
opt-level = 0
debug = true
split-debuginfo = "unpacked"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"
strip = true

[profile.bench]
inherits = "release"
debug = true
```

#### Cargo Configuration (.cargo/config.toml)
```toml
[build]
# Use all available CPU cores
jobs = 0

[target.x86_64-unknown-linux-gnu]
# Use mold linker for faster builds (Linux)
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# Enable incremental compilation for faster development
[env]
CARGO_INCREMENTAL = "1"

[alias]
# Custom cargo commands
watch-run = "watch -x 'run --bin server'"
check-all = "check --all-targets --all-features"
test-all = "test --all-targets --all-features"
```

### Dependency Categories

#### Production Dependencies
- **Web Framework**: Axum for HTTP server with async support
- **Runtime**: Tokio for async runtime with all features
- **Serialization**: Serde for JSON/binary data handling
- **Error Handling**: Anyhow and thiserror for comprehensive error management
- **Shared Library**: Local path dependency to shared Rust crate

#### Development Dependencies
- **Testing**: Criterion for benchmarking, tokio-test for async tests
- **Mocking**: Mockall for test mocks and stubs
- **Utilities**: Tempfile for test file management

#### Optional Features
- **Tracing**: Structured logging and distributed tracing
- **Metrics**: Prometheus metrics collection
- **TLS**: HTTPS support with rustls

### Package Management

#### Basic Commands
```bash
# Install/update dependencies
cargo fetch

# Build with all features
cargo build --all-features

# Run tests with all features
cargo test --all-features

# Update dependencies
cargo update

# Check for security advisories
cargo audit
```

#### Advanced Management
```bash
# Generate dependency tree
cargo tree

# Check for unused dependencies
cargo machete

# Update to latest compatible versions
cargo update --aggressive

# Clean build artifacts
cargo clean
```

## Shared Library Integration

### Cross-Language Dependency Setup

#### Python Integration
```bash
# In Python service requirements/base.txt
-e ../solidity-security-shared/python

# Verify installation
python -c "from solidity_shared import Vulnerability; print('Success')"
```

#### TypeScript Integration
```json
{
  "dependencies": {
    "@solidity-security/shared": "file:../solidity-security-shared/typescript"
  }
}
```

```bash
# Verify installation
node -e "const { Vulnerability } = require('@solidity-security/shared'); console.log('Success')"
```

#### Rust Integration
```toml
[dependencies]
solidity-security-shared = { path = "../solidity-security-shared/rust" }
```

```bash
# Verify compilation
cargo check
```

### Version Synchronization

#### Automated Version Management
```bash
#!/bin/bash
# scripts/sync-versions.sh
VERSION=$(cat VERSION)

# Update Python version
sed -i "s/version = .*/version = \"$VERSION\"/" solidity-security-shared/python/pyproject.toml

# Update TypeScript version
sed -i "s/\"version\": .*/\"version\": \"$VERSION\",/" solidity-security-shared/typescript/package.json

# Update Rust version
sed -i "s/version = .*/version = \"$VERSION\"/" solidity-security-shared/rust/Cargo.toml
```

## Dependency Security and Updates

### Security Scanning

#### Python Security
```bash
# Install security scanner
pip install safety bandit

# Scan for known vulnerabilities
safety check -r requirements/base.txt

# Code security analysis
bandit -r src/
```

#### TypeScript Security
```bash
# Audit for vulnerabilities
npm audit

# Fix automatically fixable issues
npm audit fix

# Advanced security scanning
npx audit-ci --config audit-ci.json
```

#### Rust Security
```bash
# Install cargo-audit
cargo install cargo-audit

# Scan for security advisories
cargo audit

# Scan for dependencies with known issues
cargo deny check advisories
```

### Update Strategies

#### Regular Updates
```bash
# Weekly dependency update schedule
#!/bin/bash
# scripts/weekly-update.sh

echo "Updating Python dependencies..."
pip-tools compile --upgrade requirements.in

echo "Updating TypeScript dependencies..."
npm update

echo "Updating Rust dependencies..."
cargo update

echo "Running security scans..."
safety check
npm audit
cargo audit
```

#### Breaking Change Management
1. **Test Before Update**: Run full test suite before dependency updates
2. **Incremental Updates**: Update dependencies in small batches
3. **Rollback Plan**: Maintain ability to rollback to previous versions
4. **Documentation**: Document breaking changes and migration steps

## Best Practices

### General Principles
1. **Pin Versions**: Use exact versions for production stability
2. **Regular Updates**: Schedule regular dependency updates
3. **Security First**: Prioritize security updates over feature updates
4. **Testing**: Test dependency updates in isolated environments first

### Python Best Practices
- Use virtual environments for isolation
- Split requirements by environment (base, dev, test)
- Use pip-tools for deterministic dependency resolution
- Pin both direct and transitive dependencies

### TypeScript Best Practices
- Use package-lock.json for reproducible installs
- Separate devDependencies from production dependencies
- Use workspace management for multi-package projects
- Configure strict TypeScript settings

### Rust Best Practices
- Use Cargo.lock for reproducible builds
- Enable security auditing in CI/CD
- Use feature flags for optional dependencies
- Configure profiles for different build types

This comprehensive dependency management strategy ensures consistent, secure, and maintainable dependencies across all services while supporting efficient development workflows.