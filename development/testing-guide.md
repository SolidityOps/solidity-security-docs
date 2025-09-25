# Testing and Validation Guide

## Overview

This guide covers comprehensive testing strategies across all services in the Solidity Security Platform. The testing approach spans unit testing, integration testing, performance testing, and cross-language compatibility validation.

## Testing Architecture

### Multi-Language Testing Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    Testing Pyramid                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              E2E & Integration Tests                    │   │
│  │         (Cross-service, User Workflows)                │   │
│  └─────────────────────────────────────────────────────────┘   │
│           ┌─────────────────────────────────────────┐           │
│           │           Integration Tests             │           │
│           │    (Service APIs, Database, Cache)     │           │
│           └─────────────────────────────────────────┘           │
│                  ┌─────────────────────────────┐                │
│                  │        Unit Tests           │                │
│                  │   (Functions, Components)   │                │
│                  └─────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### Testing Categories

1. **Unit Tests**: Individual functions and components
2. **Integration Tests**: Service interactions and external dependencies
3. **Performance Tests**: Load, stress, and benchmark testing
4. **Compatibility Tests**: Cross-language type and API consistency
5. **End-to-End Tests**: Complete user workflows across services

## Python Service Testing

### Testing Framework Setup

#### pytest Configuration
```python
# conftest.py - Shared test configuration
import pytest
import asyncio
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from src.main import app
from src.database import Base, get_db

# Test database setup
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture(scope="session")
def event_loop():
    """Create an instance of the default event loop for the test session."""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="function")
def db_session():
    """Create a fresh database session for each test."""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def client(db_session):
    """Create a test client with database override."""
    def override_get_db():
        try:
            yield db_session
        finally:
            db_session.close()

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as test_client:
        yield test_client
    app.dependency_overrides.clear()

@pytest.fixture
def sample_vulnerability():
    """Sample vulnerability data for tests."""
    from solidity_shared import Vulnerability, Severity
    return Vulnerability(
        title="Test Vulnerability",
        description="A test vulnerability",
        severity=Severity.HIGH,
        confidence=0.95
    )
```

#### Unit Test Examples
```python
# tests/unit/test_vulnerability_service.py
import pytest
from unittest.mock import Mock, AsyncMock
from src.services.vulnerability_service import VulnerabilityService
from solidity_shared import Vulnerability, Severity

class TestVulnerabilityService:

    @pytest.fixture
    def mock_db(self):
        return Mock()

    @pytest.fixture
    def service(self, mock_db):
        return VulnerabilityService(mock_db)

    async def test_create_vulnerability(self, service, mock_db, sample_vulnerability):
        # Arrange
        mock_db.add = Mock()
        mock_db.commit = AsyncMock()
        mock_db.refresh = AsyncMock()

        # Act
        result = await service.create_vulnerability(sample_vulnerability)

        # Assert
        assert result.title == "Test Vulnerability"
        assert result.severity == Severity.HIGH
        mock_db.add.assert_called_once()
        mock_db.commit.assert_called_once()

    async def test_calculate_risk_score(self, service):
        # Test business logic
        vuln = Vulnerability(
            severity=Severity.HIGH,
            confidence=0.9,
            title="Test"
        )

        score = service.calculate_risk_score(vuln)
        assert score > 0.7  # High severity should result in high risk

    @pytest.mark.parametrize("severity,expected_weight", [
        (Severity.CRITICAL, 1.0),
        (Severity.HIGH, 0.8),
        (Severity.MEDIUM, 0.6),
        (Severity.LOW, 0.4),
    ])
    def test_severity_weights(self, service, severity, expected_weight):
        weight = service.get_severity_weight(severity)
        assert weight == expected_weight
```

#### Integration Test Examples
```python
# tests/integration/test_api_endpoints.py
import pytest
from httpx import AsyncClient

class TestVulnerabilityAPI:

    async def test_create_vulnerability_endpoint(self, client, sample_vulnerability):
        # Test full API endpoint
        response = client.post(
            "/api/v1/vulnerabilities/",
            json=sample_vulnerability.dict()
        )

        assert response.status_code == 201
        data = response.json()
        assert data["title"] == "Test Vulnerability"
        assert data["severity"] == "HIGH"

    async def test_get_vulnerabilities_with_filters(self, client):
        # Create test data
        await self.create_test_vulnerabilities(client)

        # Test filtering
        response = client.get("/api/v1/vulnerabilities/?severity=HIGH")
        assert response.status_code == 200

        vulnerabilities = response.json()["items"]
        assert all(v["severity"] == "HIGH" for v in vulnerabilities)

    async def test_shared_library_integration(self, client):
        # Test that shared library types work correctly
        from solidity_shared import generate_contract_id, validate_vulnerability

        contract_code = "contract Test { function test() public {} }"
        contract_id = generate_contract_id(contract_code, "Test")

        assert contract_id is not None
        assert len(contract_id) > 0
```

### Performance Testing
```python
# tests/performance/test_performance.py
import pytest
import time
import asyncio
from concurrent.futures import ThreadPoolExecutor

class TestPerformance:

    @pytest.mark.benchmark
    def test_vulnerability_creation_performance(self, benchmark, service, sample_vulnerability):
        """Benchmark vulnerability creation performance."""
        result = benchmark(service.create_vulnerability, sample_vulnerability)
        assert result is not None

    async def test_concurrent_requests(self, client):
        """Test system under concurrent load."""
        async def make_request():
            response = client.get("/api/v1/health")
            return response.status_code

        # Create 100 concurrent requests
        tasks = [make_request() for _ in range(100)]
        results = await asyncio.gather(*tasks)

        # All requests should succeed
        assert all(status == 200 for status in results)

    def test_shared_library_performance(self):
        """Test shared library performance vs pure Python."""
        from solidity_shared import generate_contract_id
        import hashlib

        contract_code = "contract Test { function test() public {} }" * 100

        # Test Rust implementation
        start_time = time.time()
        for _ in range(1000):
            rust_result = generate_contract_id(contract_code, "Test")
        rust_time = time.time() - start_time

        # Test pure Python equivalent
        start_time = time.time()
        for _ in range(1000):
            python_result = hashlib.sha256(
                f"{contract_code}Test".encode()
            ).hexdigest()
        python_time = time.time() - start_time

        # Rust should be faster
        assert rust_time < python_time
        print(f"Rust: {rust_time:.3f}s, Python: {python_time:.3f}s")
```

## TypeScript Testing

### Testing Framework Setup

#### Vitest Configuration
```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    globals: true,
    coverage: {
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.ts'
      ]
    }
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@shared': path.resolve(__dirname, '../solidity-security-shared/typescript/src')
    }
  }
})
```

#### Test Setup
```typescript
// src/test/setup.ts
import '@testing-library/jest-dom'
import { vi } from 'vitest'

// Mock shared library WASM loading
vi.mock('@solidity-security/shared', async () => {
  const actual = await vi.importActual('@solidity-security/shared')
  return {
    ...actual,
    loadWasmModule: vi.fn().mockResolvedValue(null), // Use JS fallback
    isWasmAvailable: vi.fn().mockReturnValue(false)
  }
})

// Mock API calls
global.fetch = vi.fn()

// Setup test environment
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
})
```

#### Unit Test Examples
```typescript
// src/components/VulnerabilityCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { VulnerabilityCard } from './VulnerabilityCard'
import { Vulnerability, Severity } from '@solidity-security/shared'

const mockVulnerability: Vulnerability = {
  id: '123',
  title: 'Test Vulnerability',
  description: 'Test description',
  severity: Severity.High,
  confidence: 0.95,
  location: {
    file: 'test.sol',
    line: 42,
    column: 10
  }
}

describe('VulnerabilityCard', () => {
  it('renders vulnerability information correctly', () => {
    render(<VulnerabilityCard vulnerability={mockVulnerability} />)

    expect(screen.getByText('Test Vulnerability')).toBeInTheDocument()
    expect(screen.getByText('HIGH')).toBeInTheDocument()
    expect(screen.getByText('95%')).toBeInTheDocument()
  })

  it('handles severity color coding', () => {
    render(<VulnerabilityCard vulnerability={mockVulnerability} />)

    const severityBadge = screen.getByText('HIGH')
    expect(severityBadge).toHaveClass('bg-red-100', 'text-red-800')
  })

  it('calls onSelect when card is clicked', () => {
    const onSelect = vi.fn()
    render(<VulnerabilityCard vulnerability={mockVulnerability} onSelect={onSelect} />)

    fireEvent.click(screen.getByRole('article'))
    expect(onSelect).toHaveBeenCalledWith(mockVulnerability)
  })
})
```

#### Integration Test Examples
```typescript
// src/hooks/useVulnerabilities.test.ts
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useVulnerabilities } from './useVulnerabilities'
import { api } from '../services/api'

vi.mock('../services/api')

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  })

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

describe('useVulnerabilities', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('fetches vulnerabilities successfully', async () => {
    const mockData = { items: [mockVulnerability], total: 1 }
    vi.mocked(api.getVulnerabilities).mockResolvedValue(mockData)

    const { result } = renderHook(() => useVulnerabilities(), {
      wrapper: createWrapper()
    })

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true)
    })

    expect(result.current.data).toEqual(mockData)
    expect(api.getVulnerabilities).toHaveBeenCalledWith({
      page: 1,
      limit: 20
    })
  })

  it('handles API errors gracefully', async () => {
    vi.mocked(api.getVulnerabilities).mockRejectedValue(new Error('API Error'))

    const { result } = renderHook(() => useVulnerabilities(), {
      wrapper: createWrapper()
    })

    await waitFor(() => {
      expect(result.current.isError).toBe(true)
    })

    expect(result.current.error).toBeInstanceOf(Error)
  })
})
```

#### E2E Test Examples
```typescript
// e2e/dashboard.spec.ts (Playwright)
import { test, expect } from '@playwright/test'

test.describe('Dashboard', () => {
  test.beforeEach(async ({ page }) => {
    // Mock API responses
    await page.route('**/api/v1/vulnerabilities', route => {
      route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({
          items: [mockVulnerability],
          total: 1
        })
      })
    })

    await page.goto('/dashboard')
  })

  test('displays vulnerability list', async ({ page }) => {
    await expect(page.getByText('Test Vulnerability')).toBeVisible()
    await expect(page.getByText('HIGH')).toBeVisible()
  })

  test('filters vulnerabilities by severity', async ({ page }) => {
    await page.selectOption('[data-testid="severity-filter"]', 'HIGH')
    await expect(page.getByText('Test Vulnerability')).toBeVisible()

    await page.selectOption('[data-testid="severity-filter"]', 'LOW')
    await expect(page.getByText('No vulnerabilities found')).toBeVisible()
  })

  test('shared library integration works', async ({ page }) => {
    // Test that WASM/JS fallback works in browser
    const result = await page.evaluate(async () => {
      const { generateContractId } = await import('@solidity-security/shared')
      return generateContractId('contract Test {}', 'Test')
    })

    expect(result).toBeTruthy()
    expect(typeof result).toBe('string')
  })
})
```

## Rust Testing

### Testing Framework Setup

#### Unit Test Examples
```rust
// src/types/vulnerability.rs
#[cfg(test)]
mod tests {
    use super::*;
    use serde_json;

    #[test]
    fn test_vulnerability_serialization() {
        let vuln = Vulnerability {
            id: Uuid::new_v4(),
            title: "Test Vulnerability".to_string(),
            description: "Test description".to_string(),
            severity: Severity::High,
            confidence: 0.95,
            ..Default::default()
        };

        // Test JSON serialization
        let json = serde_json::to_string(&vuln).unwrap();
        let deserialized: Vulnerability = serde_json::from_str(&json).unwrap();

        assert_eq!(vuln.title, deserialized.title);
        assert_eq!(vuln.severity, deserialized.severity);
        assert_eq!(vuln.confidence, deserialized.confidence);
    }

    #[test]
    fn test_severity_ordering() {
        assert!(Severity::Critical > Severity::High);
        assert!(Severity::High > Severity::Medium);
        assert!(Severity::Medium > Severity::Low);
    }

    #[test]
    fn test_vulnerability_validation() {
        let mut vuln = Vulnerability::default();

        // Invalid confidence
        vuln.confidence = 1.5;
        assert!(vuln.validate().is_err());

        // Valid confidence
        vuln.confidence = 0.95;
        assert!(vuln.validate().is_ok());
    }
}
```

#### Integration Test Examples
```rust
// tests/integration_test.rs
use solidity_security_shared::*;
use tokio_test;

#[tokio::test]
async fn test_contract_parsing_integration() {
    let contract_code = r#"
        pragma solidity ^0.8.0;
        contract Test {
            function vulnerable() public {
                // Potential vulnerability
            }
        }
    "#;

    let contract_id = generate_contract_id(contract_code, "Test").unwrap();
    assert!(!contract_id.is_empty());

    // Test that the same input produces the same ID (deterministic)
    let contract_id2 = generate_contract_id(contract_code, "Test").unwrap();
    assert_eq!(contract_id, contract_id2);
}

#[tokio::test]
async fn test_cross_language_compatibility() {
    let vuln = Vulnerability {
        id: Uuid::new_v4(),
        title: "Cross-Language Test".to_string(),
        severity: Severity::High,
        confidence: 0.9,
        ..Default::default()
    };

    // Serialize to JSON (same format used by Python and TypeScript)
    let json = serde_json::to_string(&vuln).unwrap();

    // Deserialize back
    let deserialized: Vulnerability = serde_json::from_str(&json).unwrap();

    assert_eq!(vuln.title, deserialized.title);
    assert_eq!(vuln.severity, deserialized.severity);
}
```

#### Performance Test Examples
```rust
// benches/parsing_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use solidity_security_shared::*;

fn bench_contract_id_generation(c: &mut Criterion) {
    let contract_code = include_str!("../test-data/large-contract.sol");

    c.bench_function("generate_contract_id", |b| {
        b.iter(|| {
            generate_contract_id(
                black_box(contract_code),
                black_box("TestContract")
            )
        })
    });
}

fn bench_vulnerability_validation(c: &mut Criterion) {
    let vuln = Vulnerability {
        title: "Benchmark Test".to_string(),
        severity: Severity::High,
        confidence: 0.95,
        ..Default::default()
    };

    c.bench_function("vulnerability_validation", |b| {
        b.iter(|| {
            black_box(&vuln).validate()
        })
    });
}

criterion_group!(benches, bench_contract_id_generation, bench_vulnerability_validation);
criterion_main!(benches);
```

## Cross-Language Compatibility Testing

### Shared Library Integration Tests

#### Test Data Consistency
```python
# scripts/test_cross_language_compatibility.py
import json
import subprocess
import tempfile
from solidity_shared import Vulnerability, Severity

def test_json_compatibility():
    """Test that all three languages produce identical JSON output."""

    # Create test vulnerability in Python
    python_vuln = Vulnerability(
        title="Cross-Language Test",
        severity=Severity.HIGH,
        confidence=0.95
    )
    python_json = python_vuln.json()

    # Test TypeScript compatibility
    ts_script = f"""
    const {{ Vulnerability, Severity }} = require('@solidity-security/shared');
    const vuln = new Vulnerability({{
        title: "Cross-Language Test",
        severity: Severity.High,
        confidence: 0.95
    }});
    console.log(JSON.stringify(vuln));
    """

    with tempfile.NamedTemporaryFile(mode='w', suffix='.js', delete=False) as f:
        f.write(ts_script)
        result = subprocess.run(['node', f.name], capture_output=True, text=True)
        ts_json = result.stdout.strip()

    # Test Rust compatibility
    rust_test = f"""
    use solidity_security_shared::{{Vulnerability, Severity}};
    let vuln = Vulnerability {{
        title: "Cross-Language Test".to_string(),
        severity: Severity::High,
        confidence: 0.95,
        ..Default::default()
    }};
    println!("{{}}", serde_json::to_string(&vuln).unwrap());
    """

    # Compare JSON outputs (normalize for comparison)
    python_data = json.loads(python_json)
    ts_data = json.loads(ts_json)

    assert python_data['title'] == ts_data['title']
    assert python_data['severity'] == ts_data['severity']
    assert python_data['confidence'] == ts_data['confidence']

    print("✅ Cross-language JSON compatibility verified")

if __name__ == "__main__":
    test_json_compatibility()
```

#### Performance Comparison Tests
```python
# scripts/performance_comparison.py
import time
import statistics
from solidity_shared import generate_contract_id, RUST_AVAILABLE

def benchmark_implementation(func, *args, iterations=1000):
    """Benchmark a function implementation."""
    times = []
    for _ in range(iterations):
        start = time.perf_counter()
        result = func(*args)
        end = time.perf_counter()
        times.append(end - start)

    return {
        'mean': statistics.mean(times),
        'median': statistics.median(times),
        'min': min(times),
        'max': max(times),
        'result': result
    }

def compare_implementations():
    """Compare Rust vs Python implementations."""
    contract_code = "contract Test { function test() public {} }" * 100
    contract_name = "TestContract"

    # Benchmark current implementation (Rust if available)
    current_stats = benchmark_implementation(
        generate_contract_id, contract_code, contract_name
    )

    # Fallback to pure Python if available
    if RUST_AVAILABLE:
        import os
        os.environ['SOLIDITY_SHARED_NO_RUST'] = '1'
        # Re-import to get Python implementation
        from importlib import reload
        import solidity_shared
        reload(solidity_shared)

        python_stats = benchmark_implementation(
            solidity_shared.generate_contract_id, contract_code, contract_name
        )

        print(f"Rust implementation: {current_stats['mean']:.4f}s avg")
        print(f"Python implementation: {python_stats['mean']:.4f}s avg")
        print(f"Speedup: {python_stats['mean'] / current_stats['mean']:.2f}x")
    else:
        print(f"Python implementation: {current_stats['mean']:.4f}s avg")
        print("Rust implementation not available")

if __name__ == "__main__":
    compare_implementations()
```

## Test Automation and CI/CD

### GitHub Actions Workflows

#### Multi-Language Test Matrix
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test-python:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.11', '3.12']
        service: [
          'api-service',
          'tool-integration',
          'intelligence-engine',
          'orchestration',
          'data-service',
          'notification'
        ]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        working-directory: ./solidity-security-${{ matrix.service }}
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements/test.txt

      - name: Run tests with coverage
        working-directory: ./solidity-security-${{ matrix.service }}
        run: |
          pytest --cov=src --cov-report=xml --cov-report=html

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: python-${{ matrix.service }}

  test-typescript:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: ['18', '20']
        service: ['ui-core', 'dashboard', 'findings', 'analysis']

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        working-directory: ./solidity-security-${{ matrix.service }}
        run: npm ci

      - name: Type check
        working-directory: ./solidity-security-${{ matrix.service }}
        run: npm run type-check

      - name: Run tests
        working-directory: ./solidity-security-${{ matrix.service }}
        run: npm test -- --coverage

      - name: E2E tests
        working-directory: ./solidity-security-${{ matrix.service }}
        run: |
          npm run build
          npm run test:e2e

  test-rust:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          components: clippy, rustfmt

      - name: Cache cargo registry
        uses: actions/cache@v3
        with:
          path: ~/.cargo/registry
          key: cargo-registry-${{ hashFiles('**/Cargo.lock') }}

      - name: Check formatting
        run: cargo fmt -- --check

      - name: Run Clippy
        run: cargo clippy -- -D warnings

      - name: Run tests
        working-directory: ./solidity-security-contract-parser
        run: cargo test --all-features

      - name: Run benchmarks
        working-directory: ./solidity-security-contract-parser
        run: cargo bench

  integration-tests:
    runs-on: ubuntu-latest
    needs: [test-python, test-typescript, test-rust]

    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up test environment
        run: |
          docker-compose -f docker-compose.test.yml up -d
          sleep 30  # Wait for services to start

      - name: Run integration tests
        run: |
          python scripts/run_integration_tests.py

      - name: Run cross-language compatibility tests
        run: |
          python scripts/test_cross_language_compatibility.py
```

### Quality Gates and Checks

#### Code Quality Requirements
```yaml
# .github/workflows/quality.yml
name: Code Quality

on: [push, pull_request]

jobs:
  quality-checks:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Python Code Quality
        run: |
          # Check all Python services
          for service in api-service tool-integration intelligence-engine orchestration data-service notification; do
            echo "Checking solidity-security-$service..."
            cd "solidity-security-$service"

            # Format check
            black --check src/ tests/
            isort --check src/ tests/

            # Linting
            ruff src/ tests/

            # Type checking
            mypy src/

            cd ..
          done

      - name: TypeScript Code Quality
        run: |
          for service in ui-core dashboard findings analysis; do
            echo "Checking solidity-security-$service..."
            cd "solidity-security-$service"

            npm ci
            npm run lint
            npm run type-check

            cd ..
          done

      - name: Rust Code Quality
        run: |
          cd solidity-security-contract-parser
          cargo fmt --check
          cargo clippy -- -D warnings

      - name: Security Audit
        run: |
          # Python security
          pip install safety
          safety check -r requirements/base.txt

          # JavaScript security
          npm audit --audit-level=high

          # Rust security
          cargo install cargo-audit
          cargo audit
```

## Test Data Management

### Test Fixtures and Factories

#### Python Test Factories
```python
# tests/factories.py
import factory
from datetime import datetime
from solidity_shared import Vulnerability, Severity, Location

class VulnerabilityFactory(factory.Factory):
    class Meta:
        model = Vulnerability

    title = factory.Sequence(lambda n: f"Vulnerability {n}")
    description = factory.Faker('text', max_nb_chars=200)
    severity = factory.fuzzy.FuzzyChoice([s for s in Severity])
    confidence = factory.fuzzy.FuzzyFloat(0.1, 1.0)
    created_at = factory.LazyFunction(datetime.utcnow)

    @factory.post_generation
    def location(obj, create, extracted, **kwargs):
        if extracted:
            obj.location = extracted
        else:
            obj.location = LocationFactory()

class LocationFactory(factory.Factory):
    class Meta:
        model = Location

    file_path = factory.Faker('file_path', depth=3, extension='sol')
    start_line = factory.fuzzy.FuzzyInteger(1, 1000)
    end_line = factory.LazyAttribute(lambda obj: obj.start_line + 5)
    start_column = factory.fuzzy.FuzzyInteger(1, 80)
    end_column = factory.LazyAttribute(lambda obj: obj.start_column + 10)
```

#### TypeScript Test Data
```typescript
// src/test/factories.ts
import { Vulnerability, Severity, Location } from '@solidity-security/shared'

export const createMockVulnerability = (overrides?: Partial<Vulnerability>): Vulnerability => ({
  id: crypto.randomUUID(),
  title: 'Test Vulnerability',
  description: 'A test vulnerability for unit testing',
  severity: Severity.Medium,
  confidence: 0.8,
  location: createMockLocation(),
  created_at: new Date().toISOString(),
  updated_at: new Date().toISOString(),
  ...overrides
})

export const createMockLocation = (overrides?: Partial<Location>): Location => ({
  file_path: 'contracts/Test.sol',
  start_line: 10,
  end_line: 15,
  start_column: 5,
  end_column: 20,
  ...overrides
})

export const createMockVulnerabilities = (count: number): Vulnerability[] =>
  Array.from({ length: count }, (_, i) => createMockVulnerability({
    title: `Vulnerability ${i + 1}`,
    severity: [Severity.Low, Severity.Medium, Severity.High, Severity.Critical][i % 4]
  }))
```

## Best Practices and Guidelines

### Testing Best Practices

1. **Test Pyramid**: Focus on unit tests, with fewer integration and E2E tests
2. **Fast Feedback**: Unit tests should run quickly (< 1ms per test)
3. **Isolation**: Tests should not depend on external services or other tests
4. **Deterministic**: Tests should produce consistent results
5. **Comprehensive**: Aim for high code coverage but focus on critical paths

### Performance Testing Guidelines

1. **Baseline Establishment**: Record performance baselines for regression testing
2. **Load Testing**: Test under realistic load conditions
3. **Memory Profiling**: Monitor memory usage and potential leaks
4. **Cross-Language Comparison**: Benchmark shared library performance benefits

### Quality Assurance

1. **Code Coverage**: Maintain >80% test coverage for critical components
2. **Static Analysis**: Use linting and type checking in CI/CD
3. **Security Testing**: Regular security audits and dependency scanning
4. **Documentation**: Keep test documentation up to date with code changes

This comprehensive testing guide ensures reliable, performant, and maintainable software across all services while validating cross-language compatibility and shared library integration.