# Shared Library Build System

## Overview

The shared library uses a unified build system that supports multiple languages (Rust, Python, TypeScript) with cross-language bindings and optimized build pipelines.

## Build Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Build System Overview                    │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Rust Core     │ Python Bindings │   TypeScript/WASM      │
│                 │                 │                         │
│ cargo build     │ maturin develop │ npm run build          │
│ cargo test      │ PyO3 v0.22      │ wasm-pack build        │
│                 │                 │                         │
└─────────────────┴─────────────────┴─────────────────────────┘
```

## Prerequisites

### System Requirements
- **Rust 1.70+** with cargo
- **Python 3.13+** (latest stable recommended)
- **Node.js 18+** with npm
- **wasm-pack** for WebAssembly builds
- **maturin** for Python binding builds

### Installation
```bash
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup update stable

# Python dependencies
pip install --upgrade pip maturin

# Node.js dependencies
npm install -g wasm-pack

# Verify installations
cargo --version
python3 --version
node --version
wasm-pack --version
maturin --version
```

## Build Commands

### Rust Core Library
```bash
# Development build
cargo build

# Release build
cargo build --release

# Build with Python features
cargo build --features python --release

# Build with WASM features
cargo build --features wasm --release

# Run tests
cargo test

# Run with all features
cargo test --all-features
```

### Python Bindings
```bash
# Development environment setup
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Development build with editable install
PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1 maturin develop --features python

# Release build
PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1 maturin build --release --features python

# Wheel distribution
maturin build --release --features python --out dist/
```

### TypeScript Package
```bash
# Install dependencies
cd typescript
npm install

# Build WASM bindings
cd ../wasm-bindings
wasm-pack build --release --target bundler --out-dir ../typescript/src/wasm

# Build TypeScript
cd ../typescript
npm run build

# Run tests
npm test

# Type checking
npm run type-check
```

### WebAssembly Bindings
```bash
cd wasm-bindings

# Development build
wasm-pack build --dev --target bundler

# Production build with optimization
wasm-pack build --release --target bundler

# Build for specific targets
wasm-pack build --release --target web      # For browser <script> tags
wasm-pack build --release --target nodejs   # For Node.js environments

# Output to TypeScript directory
wasm-pack build --release --target bundler --out-dir ../typescript/src/wasm
```

## Unified Build Process

### Development Workflow
```bash
# 1. Build Rust core
cargo build --all-features

# 2. Set up Python environment
python3 -m venv .venv && source .venv/bin/activate

# 3. Build Python bindings
PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1 maturin develop --features python

# 4. Build WASM
cd wasm-bindings
wasm-pack build --release --target bundler --out-dir ../typescript/src/wasm

# 5. Build TypeScript
cd ../typescript
npm install && npm run build

# 6. Run comprehensive tests
cd .. && make test-all  # If Makefile exists
```

### Production Build
```bash
# Clean previous builds
cargo clean
rm -rf typescript/dist
rm -rf wasm-bindings/pkg

# Build all components for production
cargo build --release --all-features

# Python wheel for distribution
maturin build --release --features python --out dist/

# WASM with optimizations
cd wasm-bindings
wasm-pack build --release --target bundler --out-dir ../typescript/src/wasm

# TypeScript production build
cd ../typescript
npm run build

# Package verification
ls -la dist/
ls -la typescript/dist/
ls -la typescript/src/wasm/
```

## Build Configuration

### Rust Configuration (Cargo.toml)
```toml
[package]
name = "solidity-security-shared"
version = "0.1.0"
edition = "2021"

[lib]
name = "solidity_security_shared"
crate-type = ["cdylib", "rlib"]

[features]
default = []
python = ["pyo3"]
wasm = ["wasm-bindgen", "js-sys", "web-sys"]

[dependencies]
serde = { version = "1.0", features = ["derive"] }
uuid = { version = "1.6", features = ["v4", "serde", "js"] }
# Python bindings
pyo3 = { version = "0.22", features = ["extension-module"], optional = true }
# WASM bindings
wasm-bindgen = { version = "0.2", optional = true }
```

### Python Configuration (pyproject.toml)
```toml
[build-system]
requires = ["maturin>=1.0,<2.0"]
build-backend = "maturin"

[project]
name = "solidity-security-shared"
requires-python = ">=3.13"
dependencies = []

[tool.maturin]
features = ["pyo3/extension-module"]
python-source = "python"
```

### TypeScript Configuration (package.json)
```json
{
  "name": "solidity-security-shared",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "build:wasm": "cd ../wasm-bindings && wasm-pack build --release --target bundler --out-dir ../typescript/src/wasm",
    "dev": "tsc --watch",
    "test": "jest",
    "type-check": "tsc --noEmit"
  }
}
```

## Build Optimization

### Performance Optimization
```bash
# Rust optimizations
export RUSTFLAGS="-C target-cpu=native"
cargo build --release

# WASM size optimization
wasm-pack build --release -- --features wasm-opt

# Python wheel optimization
maturin build --release --strip
```

### Size Optimization
```bash
# Check sizes
ls -la target/release/libsolidity_security_shared.*
ls -la typescript/src/wasm/*.wasm
ls -la dist/*.whl

# WASM optimization
wasm-opt --strip-debug --strip-producers typescript/src/wasm/*.wasm
```

## Testing Strategy

### Unit Testing
```bash
# Rust tests
cargo test --all-features

# Python tests (after maturin develop)
python -m pytest python/tests/

# TypeScript tests
cd typescript && npm test
```

### Integration Testing
```bash
# Cross-language consistency test
python scripts/test_consistency.py

# WASM vs JavaScript fallback comparison
cd typescript && npm run test:integration
```

### Performance Testing
```bash
# Rust benchmarks
cargo bench

# Python performance comparison
python scripts/benchmark_python.py

# TypeScript/WASM benchmarks
cd typescript && npm run benchmark
```

## Continuous Integration

### GitHub Actions Workflow
```yaml
name: Build and Test

on: [push, pull_request]

jobs:
  rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      - run: cargo test --all-features

  python:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.13"]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install maturin
      - run: maturin develop --features python
      - run: python -m pytest

  typescript:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm install -g wasm-pack
      - run: cd wasm-bindings && wasm-pack build --release --target bundler
      - run: cd typescript && npm ci && npm test
```

## Troubleshooting

### Common Build Issues

#### Rust Compilation Errors
```bash
# Update toolchain
rustup update stable

# Clean and rebuild
cargo clean
cargo build --all-features

# Check for conflicting features
cargo tree
```

#### Python Binding Issues
```bash
# Check Python version
python3 --version  # Should be 3.13+

# Verify maturin installation
maturin --version

# Clean Python cache
rm -rf __pycache__ *.pyc

# Rebuild with verbose output
PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1 maturin develop --features python -v
```

#### WASM Build Problems
```bash
# Verify wasm-pack installation
wasm-pack --version

# Check Rust target
rustup target add wasm32-unknown-unknown

# Clean WASM artifacts
rm -rf pkg/
wasm-pack build --release --target bundler
```

#### TypeScript Issues
```bash
# Clear node modules
rm -rf node_modules package-lock.json
npm install

# Check TypeScript version
npx tsc --version

# Verify WASM files
ls -la src/wasm/
```

### Performance Issues
```bash
# Profile Rust build
cargo build --release --timings

# Check WASM size
ls -la typescript/src/wasm/*.wasm
# Expected: ~70KB

# Verify Python wheel size
ls -la dist/*.whl
# Expected: reasonable size for distribution
```

### Development Environment Setup
```bash
# Complete clean setup
git clean -fdx
cargo clean

# Rebuild everything
./scripts/build-all.sh  # If available

# Verify all components
make test-all  # If Makefile exists
```

## Distribution

### Python Package Distribution
```bash
# Build wheel
maturin build --release --features python --out dist/

# Test installation
pip install dist/*.whl

# Upload to PyPI (production)
maturin publish --features python
```

### TypeScript Package Distribution
```bash
# Build package
npm run build

# Test package
npm pack

# Publish to npm (production)
npm publish
```

### Rust Crate Distribution
```bash
# Test crate
cargo publish --dry-run

# Publish to crates.io (production)
cargo publish
```

This build system ensures reliable, reproducible builds across all supported languages while maintaining performance and compatibility.