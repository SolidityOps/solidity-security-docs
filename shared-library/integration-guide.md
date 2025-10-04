# Shared Library Integration Guide

## Quick Start Guide

This guide provides step-by-step instructions for developers to implement and use the `solidity-security-shared` library in their services.

## Installation & Setup

### Prerequisites

Ensure you have the required tools installed:

```bash
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup update stable

# Python 3.13+ (latest stable recommended)
python3 --version  # Should be 3.13+ (latest stable)
pip install --upgrade pip

# Node.js 18+
node --version     # Should be 18+
npm --version      # Should be 9+

# Additional tools
pip install maturin  # For Python-Rust bindings
npm install -g wasm-pack  # For WASM generation
```

### Development Environment Setup

```bash
# Clone the shared library repository
git clone <repository-url>
cd solidity-security-shared

# Set up development environment
make dev-setup

# Verify installation
make test
```

## Integration Examples

### Python Service Integration

#### 1. Basic Setup

```python
# requirements/base.txt
-e ../solidity-security-shared/python

# main.py
from solidity_shared import (
    Vulnerability, Finding, Analysis, Severity,
    validate_vulnerability, generate_contract_id,
    RUST_AVAILABLE
)
from fastapi import FastAPI, HTTPException
from uuid import uuid4
from datetime import datetime

app = FastAPI(title="Security Analysis Service")

print(f"Rust acceleration: {'enabled' if RUST_AVAILABLE else 'disabled'}")
```

#### 2. Creating and Validating Vulnerabilities

```python
@app.post("/vulnerabilities", response_model=Vulnerability)
async def create_vulnerability(vuln_data: dict):
    """Create a new vulnerability with validation."""
    try:
        # Create vulnerability instance
        vulnerability = Vulnerability(
            id=uuid4(),
            title=vuln_data["title"],
            description=vuln_data["description"],
            severity=Severity(vuln_data["severity"]),
            confidence=vuln_data["confidence"],
            location=vuln_data["location"],
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )

        # Validate the vulnerability
        validate_vulnerability(vulnerability)

        # Generate fingerprint for deduplication
        fingerprint = create_vulnerability_fingerprint(vulnerability)

        # Store in database (implementation specific)
        await store_vulnerability(vulnerability, fingerprint)

        return vulnerability

    except ValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail="Internal server error")
```

#### 3. Analysis Workflow Implementation

```python
@app.post("/analysis", response_model=Analysis)
async def start_analysis(contract_code: str, contract_name: str):
    """Start security analysis workflow."""
    # Generate deterministic contract ID
    contract_id = generate_contract_id(contract_code, contract_name)

    # Create analysis record
    analysis = Analysis(
        id=uuid4(),
        project_id=get_current_project_id(),  # Implementation specific
        contract_name=contract_name,
        contract_hash=hash_contract_source(contract_code),
        tools_used=["slither", "mythx", "aderyn"],
        status=AnalysisStatus.PENDING,
        started_at=datetime.utcnow(),
        metadata={
            "contract_id": str(contract_id),
            "source_lines": len(contract_code.split('\n')),
            "estimated_duration": estimate_analysis_time(contract_code)
        }
    )

    # Validate analysis data
    validate_analysis(analysis)

    # Queue for background processing
    await queue_analysis_task.delay(analysis.id, contract_code)

    return analysis
```

#### 4. Error Handling Best Practices

```python
from solidity_shared.exceptions import (
    SoliditySecurityError, ValidationError, CryptoError
)

async def safe_generate_contract_id(source: str, name: str) -> str:
    """Safely generate contract ID with error handling."""
    try:
        return generate_contract_id(source, name)
    except CryptoError as e:
        logger.error(f"Crypto operation failed: {e}")
        # Fallback to alternative ID generation
        return str(uuid5(NAMESPACE_DNS, f"{name}:{hash(source)}"))
    except Exception as e:
        logger.error(f"Unexpected error in ID generation: {e}")
        raise SoliditySecurityError("Failed to generate contract ID")

# Global exception handler
@app.exception_handler(SoliditySecurityError)
async def solidity_security_error_handler(request, exc):
    return JSONResponse(
        status_code=400,
        content={"error": "Solidity Security Error", "detail": str(exc)}
    )
```

### TypeScript Frontend Integration

#### 1. Installation and Setup

```bash
# Install the shared library
npm install @solidity-security/shared

# package.json
{
  "dependencies": {
    "@solidity-security/shared": "file:../solidity-security-shared/typescript",
    "react": "^18.2.0",
    "zod": "^3.22.0"
  }
}
```

```typescript
// src/main.tsx
import { initializeSharedLibrary } from '@solidity-security/shared';

// Initialize the library with WASM if available
initializeSharedLibrary().then(({ wasmAvailable }) => {
  console.log(`WASM acceleration: ${wasmAvailable ? 'enabled' : 'disabled'}`);
});
```

#### 2. React Component Integration

```typescript
// src/components/VulnerabilityForm.tsx
import React, { useState } from 'react';
import {
  Vulnerability, Severity, VulnerabilitySchema,
  validateVulnerability, createVulnerabilityFingerprint
} from '@solidity-security/shared';
import { z } from 'zod';

interface VulnerabilityFormProps {
  onSubmit: (vulnerability: Vulnerability) => void;
}

export const VulnerabilityForm: React.FC<VulnerabilityFormProps> = ({ onSubmit }) => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    severity: Severity.Medium,
    confidence: 0.8,
    // ... other fields
  });

  const [errors, setErrors] = useState<Record<string, string>>({});

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    try {
      // Create vulnerability object
      const vulnerability: Vulnerability = {
        id: crypto.randomUUID(),
        ...formData,
        created_at: new Date(),
        updated_at: new Date(),
      };

      // Validate with Zod schema
      VulnerabilitySchema.parse(vulnerability);

      // Additional validation with shared library
      await validateVulnerability(vulnerability);

      // Generate fingerprint
      const fingerprint = createVulnerabilityFingerprint(vulnerability);
      console.log('Vulnerability fingerprint:', fingerprint);

      // Clear errors and submit
      setErrors({});
      onSubmit(vulnerability);

    } catch (error) {
      if (error instanceof z.ZodError) {
        // Handle Zod validation errors
        const fieldErrors: Record<string, string> = {};
        error.errors.forEach(err => {
          const field = err.path.join('.');
          fieldErrors[field] = err.message;
        });
        setErrors(fieldErrors);
      } else {
        console.error('Validation failed:', error);
        setErrors({ general: 'Validation failed' });
      }
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label className="block text-sm font-medium">Title</label>
        <input
          type="text"
          value={formData.title}
          onChange={(e) => setFormData({ ...formData, title: e.target.value })}
          className="mt-1 block w-full rounded-md border-gray-300"
        />
        {errors.title && <p className="text-red-500 text-sm">{errors.title}</p>}
      </div>

      <div>
        <label className="block text-sm font-medium">Severity</label>
        <select
          value={formData.severity}
          onChange={(e) => setFormData({ ...formData, severity: e.target.value as Severity })}
          className="mt-1 block w-full rounded-md border-gray-300"
        >
          {Object.values(Severity).map(severity => (
            <option key={severity} value={severity}>{severity}</option>
          ))}
        </select>
      </div>

      {/* Additional form fields */}

      <button
        type="submit"
        className="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
      >
        Create Vulnerability
      </button>

      {errors.general && (
        <p className="text-red-500 text-sm">{errors.general}</p>
      )}
    </form>
  );
};
```

#### 3. API Integration with Type Safety

```typescript
// src/api/vulnerabilities.ts
import {
  Vulnerability, Finding, Analysis,
  VulnerabilitySchema, FindingSchema, AnalysisSchema
} from '@solidity-security/shared';

class VulnerabilitiesAPI {
  private baseURL: string;

  constructor(baseURL: string) {
    this.baseURL = baseURL;
  }

  async createVulnerability(data: Omit<Vulnerability, 'id' | 'created_at' | 'updated_at'>): Promise<Vulnerability> {
    const response = await fetch(`${this.baseURL}/vulnerabilities`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const rawData = await response.json();

    // Validate response with shared schema
    const vulnerability = VulnerabilitySchema.parse({
      ...rawData,
      created_at: new Date(rawData.created_at),
      updated_at: new Date(rawData.updated_at),
    });

    return vulnerability;
  }

  async getFindings(analysisId: string): Promise<Finding[]> {
    const response = await fetch(`${this.baseURL}/analyses/${analysisId}/findings`);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const rawData = await response.json();

    // Validate each finding
    return rawData.map((item: any) => FindingSchema.parse({
      ...item,
      created_at: new Date(item.created_at),
    }));
  }
}

export const vulnerabilitiesAPI = new VulnerabilitiesAPI(process.env.REACT_APP_API_BASE_URL!);
```

#### 4. WASM Performance Optimization

```typescript
// src/utils/performance.ts
import {
  generateContractId, hashContractSource,
  getWasmDiagnostics, WASM_AVAILABLE
} from '@solidity-security/shared';

export class ContractProcessor {
  private performanceMetrics = {
    wasmOperations: 0,
    jsOperations: 0,
    averageWasmTime: 0,
    averageJsTime: 0,
  };

  async processContract(sourceCode: string, name: string) {
    const startTime = performance.now();

    try {
      // Use WASM-accelerated operations when available
      const contractId = await generateContractId(sourceCode, name);
      const sourceHash = await hashContractSource(sourceCode);

      const endTime = performance.now();
      const duration = endTime - startTime;

      // Track performance metrics
      if (WASM_AVAILABLE) {
        this.performanceMetrics.wasmOperations++;
        this.updateAverage('wasm', duration);
      } else {
        this.performanceMetrics.jsOperations++;
        this.updateAverage('js', duration);
      }

      return { contractId, sourceHash, processingTime: duration };

    } catch (error) {
      console.error('Contract processing failed:', error);

      // Get diagnostics if WASM failed
      if (WASM_AVAILABLE) {
        const diagnostics = await getWasmDiagnostics();
        console.log('WASM diagnostics:', diagnostics);
      }

      throw error;
    }
  }

  private updateAverage(type: 'wasm' | 'js', newTime: number) {
    const key = type === 'wasm' ? 'averageWasmTime' : 'averageJsTime';
    const count = type === 'wasm' ? this.performanceMetrics.wasmOperations : this.performanceMetrics.jsOperations;

    this.performanceMetrics[key] = (this.performanceMetrics[key] * (count - 1) + newTime) / count;
  }

  getPerformanceReport() {
    return {
      ...this.performanceMetrics,
      speedupRatio: this.performanceMetrics.averageJsTime / this.performanceMetrics.averageWasmTime,
      wasmAvailable: WASM_AVAILABLE,
    };
  }
}
```

### Rust Service Integration

#### 1. Dependency Setup

```toml
# Cargo.toml
[dependencies]
solidity-security-shared = { path = "../solidity-security-shared/rust" }
axum = "0.7"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.0", features = ["v4", "serde"] }
```

#### 2. HTTP API Implementation

```rust
// src/main.rs
use axum::{
    extract::Path,
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use solidity_security_shared::{
    Vulnerability, Finding, Analysis, Severity,
    validate_vulnerability, generate_contract_id,
    SoliditySecurityError,
};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use uuid::Uuid;

#[derive(Deserialize)]
struct CreateVulnerabilityRequest {
    title: String,
    description: String,
    severity: Severity,
    confidence: f32,
    // ... other fields
}

#[derive(Serialize)]
struct ApiResponse<T> {
    success: bool,
    data: Option<T>,
    error: Option<String>,
}

async fn create_vulnerability(
    Json(request): Json<CreateVulnerabilityRequest>,
) -> Result<Json<ApiResponse<Vulnerability>>, StatusCode> {
    // Create vulnerability instance
    let vulnerability = Vulnerability {
        id: Uuid::new_v4(),
        title: request.title,
        description: request.description,
        severity: request.severity,
        confidence: request.confidence,
        // ... other fields with defaults
        ..Default::default()
    };

    // Validate the vulnerability
    match validate_vulnerability(&vulnerability) {
        Ok(_) => {
            // Store in database (implementation specific)
            // let stored = store_vulnerability(vulnerability).await?;

            Ok(Json(ApiResponse {
                success: true,
                data: Some(vulnerability),
                error: None,
            }))
        }
        Err(e) => {
            eprintln!("Validation failed: {}", e);
            Err(StatusCode::BAD_REQUEST)
        }
    }
}

async fn generate_contract_id_endpoint(
    Json(request): Json<HashMap<String, String>>,
) -> Result<Json<ApiResponse<String>>, StatusCode> {
    let source_code = request.get("source_code")
        .ok_or(StatusCode::BAD_REQUEST)?;
    let name = request.get("name")
        .ok_or(StatusCode::BAD_REQUEST)?;

    match generate_contract_id(source_code, name) {
        Ok(id) => Ok(Json(ApiResponse {
            success: true,
            data: Some(id.to_string()),
            error: None,
        })),
        Err(e) => {
            eprintln!("ID generation failed: {}", e);
            Err(StatusCode::INTERNAL_SERVER_ERROR)
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/vulnerabilities", post(create_vulnerability))
        .route("/contracts/generate-id", post(generate_contract_id_endpoint));

    println!("Starting server on http://localhost:3000");

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

#### 3. Advanced Pattern Matching

```rust
// src/pattern_matcher.rs
use solidity_security_shared::{
    Vulnerability, Finding, Severity,
    create_vulnerability_fingerprint
};
use std::collections::{HashMap, HashSet};

pub struct VulnerabilityMatcher {
    patterns: HashMap<String, Vec<VulnerabilityPattern>>,
}

#[derive(Debug, Clone)]
pub struct VulnerabilityPattern {
    pub name: String,
    pub severity: Severity,
    pub confidence_boost: f32,
    pub regex_pattern: String,
    pub swc_mappings: Vec<u32>,
}

impl VulnerabilityMatcher {
    pub fn new() -> Self {
        let mut matcher = Self {
            patterns: HashMap::new(),
        };

        // Initialize with known patterns
        matcher.load_default_patterns();
        matcher
    }

    pub fn match_vulnerabilities(&self, source_code: &str) -> Vec<Vulnerability> {
        let mut vulnerabilities = Vec::new();
        let mut seen_fingerprints = HashSet::new();

        for (category, patterns) in &self.patterns {
            for pattern in patterns {
                if let Some(vuln) = self.match_pattern(source_code, pattern, category) {
                    let fingerprint = create_vulnerability_fingerprint(&vuln);

                    // Deduplicate by fingerprint
                    if !seen_fingerprints.contains(&fingerprint) {
                        seen_fingerprints.insert(fingerprint);
                        vulnerabilities.push(vuln);
                    }
                }
            }
        }

        vulnerabilities
    }

    fn match_pattern(&self, source: &str, pattern: &VulnerabilityPattern, category: &str) -> Option<Vulnerability> {
        // Implementation of pattern matching logic
        // This would include regex matching, AST analysis, etc.

        // For demonstration purposes, simple regex matching
        let regex = regex::Regex::new(&pattern.regex_pattern).ok()?;

        if regex.is_match(source) {
            Some(Vulnerability {
                id: Uuid::new_v4(),
                title: pattern.name.clone(),
                description: format!("Pattern matched in category: {}", category),
                severity: pattern.severity,
                confidence: 0.8 + pattern.confidence_boost,
                // ... other fields
                ..Default::default()
            })
        } else {
            None
        }
    }

    fn load_default_patterns(&mut self) {
        // Load reentrancy patterns
        self.patterns.insert("reentrancy".to_string(), vec![
            VulnerabilityPattern {
                name: "Potential Reentrancy".to_string(),
                severity: Severity::High,
                confidence_boost: 0.1,
                regex_pattern: r"\.call\s*\{[^}]*\}\s*\([^)]*\)".to_string(),
                swc_mappings: vec![107], // SWC-107: Reentrancy
            },
        ]);

        // Load integer overflow patterns
        self.patterns.insert("overflow".to_string(), vec![
            VulnerabilityPattern {
                name: "Integer Overflow Risk".to_string(),
                severity: Severity::Medium,
                confidence_boost: 0.0,
                regex_pattern: r"\+\s*\+|\-\s*\-|[\w\s]*\+[\w\s]*\*".to_string(),
                swc_mappings: vec![101], // SWC-101: Integer Overflow
            },
        ]);

        // Add more patterns...
    }
}
```

## Testing Strategies

### Unit Testing with Shared Types

```python
# test_vulnerability_validation.py
import pytest
from uuid import uuid4
from datetime import datetime
from solidity_shared import (
    Vulnerability, Severity, Location,
    validate_vulnerability, ValidationError
)

def test_valid_vulnerability():
    """Test creation and validation of valid vulnerability."""
    location = Location(
        file_path="contracts/Token.sol",
        start_line=42,
        end_line=42,
        start_column=5,
        end_column=25
    )

    vuln = Vulnerability(
        id=uuid4(),
        title="Integer Overflow",
        description="Potential integer overflow in transfer function",
        severity=Severity.HIGH,
        confidence=0.85,
        location=location,
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow()
    )

    # Should not raise any exception
    validate_vulnerability(vuln)

def test_invalid_confidence_range():
    """Test validation fails for invalid confidence values."""
    vuln = create_test_vulnerability()
    vuln.confidence = 1.5  # Invalid: > 1.0

    with pytest.raises(ValidationError, match="Invalid confidence"):
        validate_vulnerability(vuln)

def test_empty_title():
    """Test validation fails for empty title."""
    vuln = create_test_vulnerability()
    vuln.title = ""

    with pytest.raises(ValidationError, match="Invalid title"):
        validate_vulnerability(vuln)
```

```typescript
// tests/validation.test.ts
import {
  Vulnerability, Severity, VulnerabilitySchema,
  validateVulnerability
} from '../src';
import { z } from 'zod';

describe('Vulnerability Validation', () => {
  test('should validate correct vulnerability', async () => {
    const vuln: Vulnerability = {
      id: crypto.randomUUID(),
      title: 'Test Vulnerability',
      description: 'Test description',
      severity: Severity.High,
      confidence: 0.9,
      location: {
        file_path: 'test.sol',
        start_line: 1,
        end_line: 1,
        start_column: 1,
        end_column: 10,
      },
      created_at: new Date(),
      updated_at: new Date(),
    };

    // Should not throw
    VulnerabilitySchema.parse(vuln);
    await validateVulnerability(vuln);
  });

  test('should reject invalid confidence values', () => {
    const vuln = createTestVulnerability();
    vuln.confidence = 1.5; // Invalid

    expect(() => VulnerabilitySchema.parse(vuln)).toThrow(z.ZodError);
  });
});
```

### Integration Testing

```python
# test_cross_language_compatibility.py
import json
import subprocess
from solidity_shared import Vulnerability, Severity

def test_python_rust_serialization_compatibility():
    """Test that Python and Rust produce identical JSON."""
    # Create vulnerability in Python
    vuln = create_test_vulnerability()
    python_json = vuln.json(sort_keys=True)

    # Serialize same data in Rust (via subprocess for demo)
    rust_result = subprocess.run([
        'cargo', 'run', '--bin', 'test-serialization',
        '--', '--json', python_json
    ], capture_output=True, text=True, cwd='../rust')

    rust_json = rust_result.stdout.strip()

    # Compare JSON strings
    assert json.loads(python_json) == json.loads(rust_json)

def test_typescript_python_schema_compatibility():
    """Test TypeScript and Python schemas are compatible."""
    vuln = create_test_vulnerability()
    python_json = vuln.json()

    # Validate with TypeScript schema (via Node.js subprocess)
    node_result = subprocess.run([
        'node', '-e', f'''
        const {{ VulnerabilitySchema }} = require('../typescript/dist');
        const data = {python_json};
        try {{
          VulnerabilitySchema.parse(data);
          console.log("VALID");
        }} catch (e) {{
          console.log("INVALID:", e.message);
        }}
        '''
    ], capture_output=True, text=True)

    assert "VALID" in node_result.stdout
```

## Performance Optimization

### Rust Performance Tips

```rust
// Use references to avoid unnecessary clones
pub fn batch_validate_vulnerabilities(vulns: &[Vulnerability]) -> Vec<Result<(), ValidationError>> {
    vulns.iter()
        .map(|vuln| validate_vulnerability(vuln))
        .collect()
}

// Use owned values only when necessary
pub fn process_vulnerabilities(vulns: Vec<Vulnerability>) -> Vec<ProcessedVulnerability> {
    vulns.into_iter()
        .filter_map(|vuln| process_single_vulnerability(vuln))
        .collect()
}

// Enable SIMD for crypto operations
#[cfg(target_arch = "x86_64")]
use sha2::Sha256;

// Use parallel processing for batch operations
use rayon::prelude::*;

pub fn parallel_fingerprint_generation(vulns: &[Vulnerability]) -> Vec<String> {
    vulns.par_iter()
        .map(create_vulnerability_fingerprint)
        .collect()
}
```

### Python Performance Optimization

```python
# Prefer Rust-accelerated functions when available
from solidity_shared import RUST_AVAILABLE

def optimized_contract_processing(contracts: List[str]) -> List[str]:
    if RUST_AVAILABLE:
        # Use batch processing in Rust for better performance
        return rust_batch_process_contracts(contracts)
    else:
        # Fall back to Python with optimizations
        return [process_single_contract(c) for c in contracts]

# Cache expensive operations
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_contract_id_generation(source_hash: str, name: str) -> str:
    return generate_contract_id(source_hash, name)

# Use async/await for I/O bound operations
async def async_vulnerability_validation(vulns: List[Vulnerability]) -> List[bool]:
    tasks = [validate_vulnerability_async(v) for v in vulns]
    return await asyncio.gather(*tasks)
```

### TypeScript Performance Optimization

```typescript
// Lazy load WASM module
let wasmModule: WasmModule | null = null;
let wasmLoadPromise: Promise<WasmModule | null> | null = null;

export function getWasmModule(): Promise<WasmModule | null> {
  if (wasmModule) return Promise.resolve(wasmModule);

  if (!wasmLoadPromise) {
    wasmLoadPromise = loadWasmModule().then(module => {
      wasmModule = module;
      return module;
    });
  }

  return wasmLoadPromise;
}

// Batch operations for better performance
export async function batchGenerateContractIds(
  contracts: Array<{ source: string; name: string }>
): Promise<string[]> {
  const wasm = await getWasmModule();

  if (wasm) {
    // Use WASM batch processing
    return contracts.map(c => wasm.generate_contract_id(c.source, c.name));
  } else {
    // JavaScript fallback with Promise.all for parallelism
    return Promise.all(
      contracts.map(c => jsGenerateContractId(c.source, c.name))
    );
  }
}

// Use Web Workers for CPU-intensive operations
export async function workerBasedProcessing(data: any[]): Promise<any[]> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('/workers/processing-worker.js');

    worker.postMessage({ type: 'PROCESS_BATCH', data });

    worker.onmessage = (e) => {
      if (e.data.type === 'BATCH_COMPLETE') {
        resolve(e.data.results);
        worker.terminate();
      }
    };

    worker.onerror = reject;
  });
}
```

This developer guide provides practical, hands-on examples for integrating the `solidity-security-shared` library into real-world applications across all three supported languages.