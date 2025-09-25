# Build Systems and Automation

## Overview

The Solidity Security Platform uses three primary build systems optimized for each language ecosystem: Python with pip/maturin, TypeScript with Vite, and Rust with Cargo. This document covers build configuration, automation, and optimization strategies across all services.

## Build System Architecture

### Multi-Language Build Strategy

```
┌─────────────────────┬─────────────────────┬─────────────────────┐
│     Python          │    TypeScript       │       Rust          │
│   (6 services)      │   (4 services)      │   (1 service)       │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ • pip/setuptools    │ • Vite + esbuild    │ • Cargo             │
│ • maturin (PyO3)    │ • TypeScript 5+     │ • rustc stable      │
│ • pytest           │ • Vitest/Jest       │ • cargo-test        │
│ • black/isort       │ • ESLint/Prettier   │ • rustfmt/clippy    │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

### Shared Library Integration

The build systems coordinate through the shared library:

```
solidity-security-shared/
├── rust/           # Core Rust library
│   └── Cargo.toml  # Rust build configuration
├── python/         # PyO3 Python bindings
│   ├── setup.py    # maturin build setup
│   └── pyproject.toml
├── typescript/     # WASM TypeScript bindings
│   ├── package.json
│   └── vite.config.ts
└── Makefile        # Unified build orchestration
```

## Python Build System

### Core Configuration

#### requirements/ Structure
```
requirements/
├── base.txt        # Production dependencies
├── dev.txt         # Development dependencies
└── test.txt        # Testing dependencies
```

#### pyproject.toml Configuration
```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "solidity-security-[service-name]"
dynamic = ["version"]
description = "Service description"
requires-python = ">=3.11"

[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'

[tool.isort]
profile = "black"
multi_line_output = 3
line_length = 88

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
addopts = "-v --tb=short"
```

### Build Commands

#### Development Builds
```bash
# Install in development mode
pip install -e .

# Install with development dependencies
pip install -r requirements/dev.txt

# Install shared library in development mode
pip install -e ../solidity-security-shared/python
```

#### Production Builds
```bash
# Build distribution packages
python -m build

# Install production dependencies only
pip install -r requirements/base.txt
```

#### PyO3 Integration (Shared Library)
```bash
# Build Python-Rust bindings with maturin
cd solidity-security-shared/python
maturin develop  # Development build
maturin build    # Release build
```

### Service-Specific Build Patterns

#### FastAPI Services (API, Data, Notification)
```bash
# Development server with hot reload
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000

# Production server
gunicorn src.main:app -w 4 -k uvicorn.workers.UvicornWorker
```

#### Celery Services (Orchestration, Tool Integration)
```bash
# Start Celery worker
celery -A src.celery:celery_app worker --loglevel=info

# Start Celery beat scheduler
celery -A src.celery:celery_app beat --loglevel=info
```

#### ML Services (Intelligence Engine)
```bash
# Install ML dependencies
pip install -r requirements/ml.txt

# Build with GPU support (optional)
pip install tensorflow[gpu] torch torchvision
```

## TypeScript Build System

### Vite Configuration

#### vite.config.ts
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@shared': path.resolve(__dirname, '../solidity-security-shared/typescript/src')
    }
  },
  server: {
    port: 3000,
    host: true,
    hmr: {
      overlay: true
    }
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          ui: ['@headlessui/react', '@heroicons/react']
        }
      }
    }
  },
  optimizeDeps: {
    include: ['@solidity-security/shared']
  }
})
```

#### package.json Scripts
```json
{
  "scripts": {
    "dev": "vite --port 3000",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext ts,tsx --fix",
    "type-check": "tsc --noEmit",
    "format": "prettier --write \"src/**/*.{ts,tsx}\""
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
      "@shared/*": ["../solidity-security-shared/typescript/src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### Build Commands

#### Development Builds
```bash
# Start development server with HMR
npm run dev

# Type checking in watch mode
npm run type-check -- --watch
```

#### Production Builds
```bash
# Full production build
npm run build

# Preview production build locally
npm run preview
```

#### Testing
```bash
# Run unit tests
npm test

# Run tests with UI
npm run test:ui

# Run tests with coverage
npm run test -- --coverage
```

### Service-Specific Build Patterns

#### UI Core (Shared Components)
```json
{
  "name": "@solidity-security/ui-core",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsc && vite build --mode lib"
  }
}
```

#### Application Services (Dashboard, Findings, Analysis)
```bash
# Build with environment-specific configurations
npm run build -- --mode development
npm run build -- --mode staging
npm run build -- --mode production
```

## Rust Build System

### Cargo Configuration

#### Cargo.toml
```toml
[package]
name = "solidity-security-contract-parser"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"

[[bin]]
name = "server"
path = "src/main.rs"

[dependencies]
# HTTP server
axum = "0.7"
tokio = { version = "1.35", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Shared library
solidity-security-shared = { path = "../solidity-security-shared/rust" }

[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "parsing_benchmark"
harness = false

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"

[profile.dev]
opt-level = 0
debug = true
```

### Build Commands

#### Development Builds
```bash
# Fast development build
cargo build

# Check compilation without building
cargo check

# Build and run
cargo run

# Watch for changes (requires cargo-watch)
cargo watch -x run
```

#### Production Builds
```bash
# Optimized release build
cargo build --release

# Run release binary
cargo run --release
```

#### Testing and Quality
```bash
# Run unit tests
cargo test

# Run with output
cargo test -- --nocapture

# Benchmark tests
cargo bench

# Format code
cargo fmt

# Lint with Clippy
cargo clippy -- -D warnings
```

### Performance Optimization

#### Build-time Optimizations
```toml
# Cargo.toml optimizations
[profile.release]
opt-level = 3          # Maximum optimization
lto = true             # Link-time optimization
codegen-units = 1      # Better optimization, slower builds
panic = "abort"        # Smaller binary size
strip = true           # Remove debug symbols
```

#### Compile-time Features
```toml
[features]
default = ["tracing"]
tracing = ["dep:tracing"]
metrics = ["dep:prometheus"]
```

## Unified Build Orchestration

### Makefile Integration

The shared library provides a unified Makefile for coordinating builds:

```makefile
# Root Makefile in solidity-security-shared/
.PHONY: all build test clean install-deps

all: build test

install-deps:
	@echo "Installing Rust dependencies..."
	cargo fetch
	@echo "Installing Python dependencies..."
	cd python && pip install -e .
	@echo "Installing TypeScript dependencies..."
	cd typescript && npm install

build:
	@echo "Building Rust core..."
	cargo build --release
	@echo "Building Python bindings..."
	cd python && maturin develop
	@echo "Building WASM bindings..."
	cd wasm-bindings && wasm-pack build --target bundler
	@echo "Building TypeScript package..."
	cd typescript && npm run build

test:
	@echo "Testing Rust core..."
	cargo test
	@echo "Testing Python bindings..."
	cd python && python -m pytest
	@echo "Testing TypeScript bindings..."
	cd typescript && npm test

clean:
	cargo clean
	cd python && rm -rf build/ dist/ target/
	cd typescript && rm -rf dist/ node_modules/.cache
	cd wasm-bindings && rm -rf pkg/

format:
	cargo fmt
	cd python && black . && isort .
	cd typescript && npm run format

lint:
	cargo clippy -- -D warnings
	cd python && ruff check .
	cd typescript && npm run lint
```

### CI/CD Integration

#### GitHub Actions Workflow
```yaml
name: Build and Test
on: [push, pull_request]

jobs:
  rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      - run: cargo build --all-features
      - run: cargo test --all-features

  python:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.11', '3.12']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements/dev.txt
      - run: pytest

  typescript:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run type-check
      - run: npm run build
      - run: npm test
```

## Build Optimization Strategies

### Parallel Builds

#### Multi-core Compilation
```bash
# Rust parallel builds
export CARGO_BUILD_JOBS=$(nproc)
cargo build

# Python parallel installs
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt --upgrade --force-reinstall
```

#### Concurrent Service Builds
```bash
# Build multiple services simultaneously
make build-python &
make build-typescript &
make build-rust &
wait
```

### Incremental Builds

#### Rust Incremental Compilation
```toml
# .cargo/config.toml
[build]
incremental = true
```

#### TypeScript Project References
```json
{
  "references": [
    { "path": "../solidity-security-ui-core" },
    { "path": "../solidity-security-shared/typescript" }
  ]
}
```

### Build Caching

#### Cargo Caching
```bash
# Use sccache for Rust build caching
export RUSTC_WRAPPER=sccache
cargo build
```

#### NPM/Yarn Caching
```bash
# Use npm cache effectively
npm ci --cache .npm

# Or use yarn with proper cache
yarn install --frozen-lockfile
```

#### Docker Build Caching
```dockerfile
# Multi-stage builds with layer caching
FROM rust:1.75 as rust-builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN cargo fetch
COPY src ./src
RUN cargo build --release

FROM node:18 as node-builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
```

## Troubleshooting Build Issues

### Common Build Problems

#### Python Build Issues
```bash
# Missing system dependencies
sudo apt-get install build-essential python3-dev

# PyO3 compilation errors
pip install maturin
maturin develop --release

# Virtual environment conflicts
rm -rf venv/
python3 -m venv venv --clear
```

#### TypeScript Build Issues
```bash
# Type errors
npm run type-check

# Module resolution issues
rm -rf node_modules/ package-lock.json
npm install

# Memory issues during build
export NODE_OPTIONS="--max-old-space-size=4096"
npm run build
```

#### Rust Build Issues
```bash
# Dependency conflicts
cargo clean
cargo update

# Missing system libraries
sudo apt-get install pkg-config libssl-dev

# Linking errors
export PKG_CONFIG_PATH="/usr/lib/pkgconfig"
```

### Performance Debugging

#### Build Time Analysis
```bash
# Rust build timing
cargo build --timings

# TypeScript bundle analysis
npm run build -- --analyze

# Python install timing
pip install --verbose -r requirements.txt
```

#### Resource Monitoring
```bash
# Monitor build resource usage
htop

# Check disk space
df -h

# Monitor memory usage during builds
watch -n 1 'free -h && ps aux --sort=-%mem | head'
```

## Best Practices

### Development Workflow
1. **Incremental Development**: Use development builds for faster iteration
2. **Hot Reload**: Configure hot reload for all applicable services
3. **Type Safety**: Run type checking continuously during development
4. **Testing**: Run relevant tests before commits

### Production Builds
1. **Optimization**: Use release/production configurations
2. **Testing**: Run full test suites before deployment
3. **Caching**: Implement effective caching strategies
4. **Monitoring**: Track build times and success rates

### Dependency Management
1. **Lock Files**: Commit lock files for reproducible builds
2. **Regular Updates**: Keep dependencies updated with security patches
3. **Conflict Resolution**: Handle version conflicts promptly
4. **Minimal Dependencies**: Only include necessary dependencies

This comprehensive build system configuration ensures consistent, efficient, and reliable builds across all services while maintaining optimal development experience and production performance.