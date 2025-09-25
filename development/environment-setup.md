# Development Environment Setup

## Overview

This guide covers setting up development environments across all services in the Solidity Security Platform. The platform consists of 11 services using Python, TypeScript/React, and Rust, all integrated with a shared multi-language library.

## Service Architecture

### Backend Services (Python)
- **API Service** - FastAPI gateway and main REST API
- **Tool Integration Service** - Multi-tool orchestration and execution
- **Intelligence Engine** - AI/ML processing and analysis
- **Orchestration Service** - Celery workers and task distribution
- **Data Service** - Database operations and caching
- **Notification Service** - WebSocket, email, and external integrations

### Frontend Services (TypeScript/React)
- **UI Core** - Shared React components and utilities
- **Dashboard** - Main user interface and navigation
- **Findings** - Vulnerability findings management interface
- **Analysis** - Analysis workflow and results interface

### Parser Service (Rust)
- **Contract Parser** - High-performance Solidity contract parsing

## Prerequisites

### System Requirements

```bash
# Rust toolchain (latest stable)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup update stable

# Python 3.11+
python3 --version  # Should be 3.11+
pip install --upgrade pip

# Node.js 18 LTS
node --version     # Should be 18+
npm --version      # Should be 9+

# Additional build tools
pip install maturin  # For Python-Rust bindings
npm install -g wasm-pack  # For WASM generation
```

### Database and Cache
```bash
# PostgreSQL 12+ (for data persistence)
# Redis 6+ (for caching and queues)
# Optional: Elasticsearch 7+ (for search)
```

## Quick Start

### 1. Clone All Repositories
```bash
# Create workspace directory
mkdir -p ~/solidity-security-platform
cd ~/solidity-security-platform

# Clone all service repositories
git clone <api-service-url> solidity-security-api-service
git clone <tool-integration-url> solidity-security-tool-integration
git clone <intelligence-engine-url> solidity-security-intelligence-engine
git clone <orchestration-url> solidity-security-orchestration
git clone <data-service-url> solidity-security-data-service
git clone <notification-url> solidity-security-notification
git clone <ui-core-url> solidity-security-ui-core
git clone <dashboard-url> solidity-security-dashboard
git clone <findings-url> solidity-security-findings
git clone <analysis-url> solidity-security-analysis
git clone <parser-url> solidity-security-contract-parser
git clone <shared-library-url> solidity-security-shared
```

### 2. Build Shared Library
```bash
cd solidity-security-shared

# Install all dependencies
make install-deps

# Build all language bindings
make build

# Run tests to verify setup
make test
```

### 3. Set Up Each Service Category

#### Python Services Setup
```bash
# For each Python service
cd solidity-security-[service-name]

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements/dev.txt

# Run basic tests
pytest tests/ -v
```

#### TypeScript Services Setup
```bash
# For each TypeScript service
cd solidity-security-[service-name]

# Install dependencies
npm install

# Run TypeScript compilation check
npm run type-check

# Start development server
npm run dev
```

#### Rust Service Setup
```bash
cd solidity-security-contract-parser

# Build and test
cargo build
cargo test

# Run development server
cargo run
```

## Environment Configuration

### Common Environment Variables

Each service requires a `.env` file based on the provided `.env.example`:

```bash
# Copy template and customize
cp .env.example .env
```

#### Shared Configuration Patterns
```bash
# Database connections
DATABASE_URL="postgresql://user:pass@localhost/solidity_security"
REDIS_URL="redis://localhost:6379/0"

# Service discovery
API_SERVICE_URL="http://localhost:8000"
DATA_SERVICE_URL="http://localhost:8002"
NOTIFICATION_SERVICE_URL="http://localhost:8003"

# Development settings
DEBUG=true
LOG_LEVEL=DEBUG
HOT_RELOAD=true
```

### Service-Specific Ports
- **API Service**: 8000
- **Intelligence Engine**: 8001
- **Data Service**: 8002
- **Notification Service**: 8003
- **Tool Integration**: 8004
- **Orchestration Service**: 8005
- **UI Core**: 3000
- **Dashboard**: 3001
- **Findings**: 3002
- **Analysis**: 3003
- **Contract Parser**: 8010

## Development Workflow

### Starting Services

#### Backend Services (Python)
```bash
# Start each Python service
cd solidity-security-[service-name]
source venv/bin/activate
uvicorn src.main:app --reload --port [SERVICE_PORT]

# Or for Celery workers
celery -A src.celery:celery_app worker --loglevel=info
```

#### Frontend Services (TypeScript)
```bash
# Start each React service
cd solidity-security-[service-name]
npm run dev  # Starts Vite dev server with hot reload
```

#### Parser Service (Rust)
```bash
cd solidity-security-contract-parser
cargo run  # Starts HTTP server on port 8010
```

### Hot Reload and Development

- **Python**: FastAPI with `--reload` provides automatic restarts on code changes
- **TypeScript**: Vite provides instant hot module replacement (HMR)
- **Rust**: `cargo watch` can be used for automatic rebuilds on changes

```bash
# Install cargo-watch for Rust hot reload
cargo install cargo-watch

# Run with auto-reload
cargo watch -x run
```

## Testing and Validation

### Running Tests

#### Python Services
```bash
# Unit tests
pytest tests/ -v

# With coverage
pytest --cov=src --cov-report=html

# Specific test categories
pytest -m unit        # Unit tests only
pytest -m integration # Integration tests only
```

#### TypeScript Services
```bash
# Unit and component tests
npm test

# E2E tests (if configured)
npm run test:e2e

# Type checking
npm run type-check
```

#### Rust Service
```bash
# Unit tests
cargo test

# With output
cargo test -- --nocapture

# Benchmark tests
cargo bench
```

### Integration Testing

#### Shared Library Integration
```bash
# Test Python imports
python -c "from solidity_shared import Vulnerability, Severity; print('Python OK')"

# Test TypeScript imports
node -e "const { Vulnerability } = require('@solidity-security/shared'); console.log('TypeScript OK')"

# Test Rust imports
cd solidity-security-contract-parser
cargo check  # Verifies shared library compiles
```

#### Cross-Service Communication
```bash
# Start all services and test API connectivity
curl http://localhost:8000/health  # API Service
curl http://localhost:8002/health  # Data Service
curl http://localhost:8003/health  # Notification Service
```

## Code Quality and Formatting

### Python Services
```bash
# Format code
black src/ tests/
isort src/ tests/

# Lint code
ruff src/ tests/

# Type checking
mypy src/
```

### TypeScript Services
```bash
# Format and lint
npm run format
npm run lint

# Fix auto-fixable issues
npm run lint:fix
```

### Rust Service
```bash
# Format code
cargo fmt

# Lint code
cargo clippy -- -D warnings

# Check without building
cargo check
```

## Troubleshooting

### Common Issues

#### Python Environment Issues
```bash
# Virtual environment conflicts
deactivate
rm -rf venv/
python3 -m venv venv
source venv/bin/activate

# Shared library import errors
pip install -e ../solidity-security-shared/python
```

#### TypeScript Build Issues
```bash
# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# TypeScript compilation errors
npm run type-check
```

#### Rust Compilation Issues
```bash
# Clean build
cargo clean
cargo build

# Update dependencies
cargo update
```

### Performance Optimization

#### Development Build Times
- Use `cargo check` instead of `cargo build` for faster feedback
- Enable TypeScript incremental compilation
- Use Python `pip install -e .` for editable installs

#### Resource Usage
- Limit concurrent service startup to avoid memory issues
- Use Docker Compose for consistent database/cache setup
- Configure IDE to exclude `node_modules/` and `target/` directories

## IDE and Editor Setup

### Recommended Extensions

#### Visual Studio Code
```json
{
  "recommendations": [
    "ms-python.python",
    "ms-python.black-formatter",
    "bradlc.vscode-tailwindcss",
    "rust-lang.rust-analyzer",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

#### Development Settings
- Enable format on save
- Configure path mappings for shared library imports
- Set up debugging configurations for each service type

## Next Steps

1. **Service Integration**: Configure service-to-service communication
2. **Database Setup**: Initialize shared database schemas
3. **Authentication**: Set up development authentication flow
4. **Monitoring**: Add development logging and metrics
5. **Docker Setup**: Optional containerized development environment

## Additional Resources

- [Shared Library Documentation](../shared-library/README.md)
- [API Reference](../shared-library/api-reference.md)
- [Integration Guide](../shared-library/integration-guide.md)
- Service-specific README files in each repository