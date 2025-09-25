# Shared Library API Reference

## Overview

This document provides detailed technical specifications for the `solidity-security-shared` library, including API references, configuration options, and integration patterns.

## API Reference

### Rust Core API

#### Types Module

```rust
// Core vulnerability structure
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Vulnerability {
    pub id: Uuid,
    pub swc_id: Option<String>,
    pub title: String,
    pub description: String,
    pub severity: Severity,
    pub confidence: f32, // 0.0 to 1.0
    pub location: Location,
    pub source_code: Option<String>,
    pub line_number: Option<u32>,
    pub column_number: Option<u32>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Security severity levels
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum Severity {
    Critical,
    High,
    Medium,
    Low,
    Informational,
}

// Source code location
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Location {
    pub file_path: String,
    pub start_line: u32,
    pub end_line: u32,
    pub start_column: u32,
    pub end_column: u32,
}

// Analysis finding structure
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Finding {
    pub id: Uuid,
    pub analysis_id: Uuid,
    pub vulnerability_id: Option<Uuid>,
    pub tool_name: String,
    pub raw_output: String,
    pub normalized_severity: Severity,
    pub confidence_score: f32,
    pub false_positive_score: f32,
    pub metadata: HashMap<String, Value>,
    pub created_at: DateTime<Utc>,
}

// Analysis run metadata
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Analysis {
    pub id: Uuid,
    pub project_id: Uuid,
    pub contract_name: String,
    pub contract_hash: String,
    pub tools_used: Vec<String>,
    pub status: AnalysisStatus,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub metadata: HashMap<String, Value>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum AnalysisStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
}
```

#### Validation API

```rust
// Validation functions
pub fn validate_vulnerability(vuln: &Vulnerability) -> Result<(), ValidationError> {
    // Validate title length (1-200 characters)
    if vuln.title.is_empty() || vuln.title.len() > 200 {
        return Err(ValidationError::InvalidTitle);
    }

    // Validate description length (1-10000 characters)
    if vuln.description.is_empty() || vuln.description.len() > 10000 {
        return Err(ValidationError::InvalidDescription);
    }

    // Validate confidence range (0.0-1.0)
    if vuln.confidence < 0.0 || vuln.confidence > 1.0 {
        return Err(ValidationError::InvalidConfidence);
    }

    Ok(())
}

pub fn validate_finding(finding: &Finding) -> Result<(), ValidationError> {
    // Validate tool name
    if finding.tool_name.is_empty() {
        return Err(ValidationError::EmptyToolName);
    }

    // Validate confidence score
    if finding.confidence_score < 0.0 || finding.confidence_score > 1.0 {
        return Err(ValidationError::InvalidConfidenceScore);
    }

    Ok(())
}

pub fn validate_analysis(analysis: &Analysis) -> Result<(), ValidationError> {
    // Validate contract name
    if analysis.contract_name.is_empty() {
        return Err(ValidationError::EmptyContractName);
    }

    // Validate contract hash format
    if !is_valid_hash(&analysis.contract_hash) {
        return Err(ValidationError::InvalidContractHash);
    }

    Ok(())
}
```

#### Crypto API

```rust
// Cryptographic utilities
pub fn generate_contract_id(source_code: &str, name: &str) -> Result<Uuid, CryptoError> {
    let combined = format!("{}:{}", name, source_code);
    let hash = sha256::digest(combined.as_bytes());
    let uuid = Uuid::new_v5(&Uuid::NAMESPACE_DNS, hash.as_bytes());
    Ok(uuid)
}

pub fn hash_contract_source(source: &str) -> String {
    sha256::digest(source.as_bytes())
}

pub fn create_vulnerability_fingerprint(vuln: &Vulnerability) -> String {
    let data = format!(
        "{}:{}:{}:{}",
        vuln.title,
        vuln.location.file_path,
        vuln.location.start_line,
        vuln.severity as u8
    );
    sha256::digest(data.as_bytes())
}

pub fn calculate_risk_score(
    severity: Severity,
    confidence: f32,
    context: &RiskContext,
) -> f32 {
    let severity_weight = match severity {
        Severity::Critical => 1.0,
        Severity::High => 0.8,
        Severity::Medium => 0.6,
        Severity::Low => 0.4,
        Severity::Informational => 0.2,
    };

    let base_score = severity_weight * confidence;

    // Apply context multipliers
    let context_multiplier = calculate_context_multiplier(context);

    (base_score * context_multiplier).min(1.0)
}
```

#### Constants API

```rust
// Severity weights for scoring
pub const SEVERITY_WEIGHTS: [(Severity, f32); 5] = [
    (Severity::Critical, 10.0),
    (Severity::High, 8.0),
    (Severity::Medium, 6.0),
    (Severity::Low, 4.0),
    (Severity::Informational, 2.0),
];

// SWC (Smart Contract Weakness Classification) mappings
pub const SWC_MAPPINGS: &[(u32, &str)] = &[
    (100, "Function Default Visibility"),
    (101, "Integer Overflow and Underflow"),
    (102, "Outdated Compiler Version"),
    (103, "Floating Pragma"),
    (104, "Unchecked Call Return Value"),
    (105, "Unprotected Ether Withdrawal"),
    (106, "Unprotected SELFDESTRUCT Instruction"),
    (107, "Reentrancy"),
    (108, "State Variable Default Visibility"),
    (109, "Uninitialized Storage Pointer"),
    // ... additional mappings
];

// Default configuration values
pub const DEFAULT_CONFIDENCE_THRESHOLD: f32 = 0.7;
pub const DEFAULT_MAX_FINDINGS_PER_ANALYSIS: usize = 1000;
pub const DEFAULT_ANALYSIS_TIMEOUT_SECONDS: u64 = 300;
```

### Python API Reference

#### Pydantic Models

```python
from pydantic import BaseModel, Field, validator
from enum import Enum
from typing import Optional, Dict, Any, List
from datetime import datetime
from uuid import UUID

class Severity(str, Enum):
    CRITICAL = "CRITICAL"
    HIGH = "HIGH"
    MEDIUM = "MEDIUM"
    LOW = "LOW"
    INFORMATIONAL = "INFORMATIONAL"

class AnalysisStatus(str, Enum):
    PENDING = "PENDING"
    RUNNING = "RUNNING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    CANCELLED = "CANCELLED"

class Location(BaseModel):
    file_path: str = Field(..., min_length=1)
    start_line: int = Field(..., ge=1)
    end_line: int = Field(..., ge=1)
    start_column: int = Field(..., ge=1)
    end_column: int = Field(..., ge=1)

    @validator('end_line')
    def end_line_must_be_gte_start_line(cls, v, values):
        if 'start_line' in values and v < values['start_line']:
            raise ValueError('end_line must be >= start_line')
        return v

class Vulnerability(BaseModel):
    id: UUID
    swc_id: Optional[str] = None
    title: str = Field(..., min_length=1, max_length=200)
    description: str = Field(..., min_length=1, max_length=10000)
    severity: Severity
    confidence: float = Field(..., ge=0.0, le=1.0)
    location: Location
    source_code: Optional[str] = None
    line_number: Optional[int] = Field(None, ge=1)
    column_number: Optional[int] = Field(None, ge=1)
    created_at: datetime
    updated_at: datetime

    class Config:
        use_enum_values = True
        json_encoders = {
            UUID: str,
            datetime: lambda dt: dt.isoformat(),
        }

class Finding(BaseModel):
    id: UUID
    analysis_id: UUID
    vulnerability_id: Optional[UUID] = None
    tool_name: str = Field(..., min_length=1)
    raw_output: str
    normalized_severity: Severity
    confidence_score: float = Field(..., ge=0.0, le=1.0)
    false_positive_score: float = Field(..., ge=0.0, le=1.0)
    metadata: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime

class Analysis(BaseModel):
    id: UUID
    project_id: UUID
    contract_name: str = Field(..., min_length=1)
    contract_hash: str = Field(..., regex=r'^[a-fA-F0-9]{64}$')
    tools_used: List[str]
    status: AnalysisStatus
    started_at: datetime
    completed_at: Optional[datetime] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)
```

#### Validation Functions

```python
def validate_vulnerability(vuln: Vulnerability) -> None:
    """Validate vulnerability data structure."""
    # Additional validation beyond Pydantic
    if vuln.swc_id and not vuln.swc_id.startswith('SWC-'):
        raise ValueError("SWC ID must start with 'SWC-'")

    if vuln.line_number and vuln.line_number <= 0:
        raise ValueError("Line number must be positive")

def validate_finding(finding: Finding) -> None:
    """Validate finding data structure."""
    if finding.tool_name not in SUPPORTED_TOOLS:
        raise ValueError(f"Unsupported tool: {finding.tool_name}")

def validate_analysis(analysis: Analysis) -> None:
    """Validate analysis data structure."""
    if analysis.completed_at and analysis.completed_at < analysis.started_at:
        raise ValueError("Completion time cannot be before start time")
```

#### Utility Functions

```python
def generate_contract_id(source_code: str, name: str) -> UUID:
    """Generate deterministic contract ID."""
    if RUST_AVAILABLE:
        return rust_generate_contract_id(source_code, name)
    else:
        return python_generate_contract_id(source_code, name)

def hash_contract_source(source: str) -> str:
    """Generate SHA-256 hash of contract source."""
    import hashlib
    return hashlib.sha256(source.encode('utf-8')).hexdigest()

def create_vulnerability_fingerprint(vuln: Vulnerability) -> str:
    """Create unique fingerprint for vulnerability."""
    data = f"{vuln.title}:{vuln.location.file_path}:{vuln.location.start_line}:{vuln.severity}"
    return hashlib.sha256(data.encode('utf-8')).hexdigest()

def calculate_risk_score(
    severity: Severity,
    confidence: float,
    context: Optional[Dict[str, Any]] = None
) -> float:
    """Calculate risk score for vulnerability."""
    severity_weights = {
        Severity.CRITICAL: 1.0,
        Severity.HIGH: 0.8,
        Severity.MEDIUM: 0.6,
        Severity.LOW: 0.4,
        Severity.INFORMATIONAL: 0.2,
    }

    base_score = severity_weights[severity] * confidence

    if context:
        # Apply context multipliers
        base_score *= _calculate_context_multiplier(context)

    return min(base_score, 1.0)
```

### TypeScript API Reference

#### Type Definitions

```typescript
// Enums
export enum Severity {
  Critical = "CRITICAL",
  High = "HIGH",
  Medium = "MEDIUM",
  Low = "LOW",
  Informational = "INFORMATIONAL"
}

export enum AnalysisStatus {
  Pending = "PENDING",
  Running = "RUNNING",
  Completed = "COMPLETED",
  Failed = "FAILED",
  Cancelled = "CANCELLED"
}

// Interfaces
export interface Location {
  file_path: string;
  start_line: number;
  end_line: number;
  start_column: number;
  end_column: number;
}

export interface Vulnerability {
  id: string;
  swc_id?: string;
  title: string;
  description: string;
  severity: Severity;
  confidence: number;
  location: Location;
  source_code?: string;
  line_number?: number;
  column_number?: number;
  created_at: Date;
  updated_at: Date;
}

export interface Finding {
  id: string;
  analysis_id: string;
  vulnerability_id?: string;
  tool_name: string;
  raw_output: string;
  normalized_severity: Severity;
  confidence_score: number;
  false_positive_score: number;
  metadata: Record<string, any>;
  created_at: Date;
}

export interface Analysis {
  id: string;
  project_id: string;
  contract_name: string;
  contract_hash: string;
  tools_used: string[];
  status: AnalysisStatus;
  started_at: Date;
  completed_at?: Date;
  metadata: Record<string, any>;
}
```

#### Zod Validation Schemas

```typescript
import { z } from 'zod';

export const LocationSchema = z.object({
  file_path: z.string().min(1),
  start_line: z.number().int().min(1),
  end_line: z.number().int().min(1),
  start_column: z.number().int().min(1),
  end_column: z.number().int().min(1),
}).refine((data) => data.end_line >= data.start_line, {
  message: "end_line must be >= start_line",
});

export const VulnerabilitySchema = z.object({
  id: z.string().uuid(),
  swc_id: z.string().optional(),
  title: z.string().min(1).max(200),
  description: z.string().min(1).max(10000),
  severity: z.nativeEnum(Severity),
  confidence: z.number().min(0).max(1),
  location: LocationSchema,
  source_code: z.string().optional(),
  line_number: z.number().int().min(1).optional(),
  column_number: z.number().int().min(1).optional(),
  created_at: z.date(),
  updated_at: z.date(),
});

export const FindingSchema = z.object({
  id: z.string().uuid(),
  analysis_id: z.string().uuid(),
  vulnerability_id: z.string().uuid().optional(),
  tool_name: z.string().min(1),
  raw_output: z.string(),
  normalized_severity: z.nativeEnum(Severity),
  confidence_score: z.number().min(0).max(1),
  false_positive_score: z.number().min(0).max(1),
  metadata: z.record(z.any()),
  created_at: z.date(),
});

export const AnalysisSchema = z.object({
  id: z.string().uuid(),
  project_id: z.string().uuid(),
  contract_name: z.string().min(1),
  contract_hash: z.string().regex(/^[a-fA-F0-9]{64}$/),
  tools_used: z.array(z.string()),
  status: z.nativeEnum(AnalysisStatus),
  started_at: z.date(),
  completed_at: z.date().optional(),
  metadata: z.record(z.any()),
});
```

#### Utility Functions

```typescript
export async function generateContractId(
  sourceCode: string,
  name: string
): Promise<string> {
  if (WASM_AVAILABLE) {
    return await wasmGenerateContractId(sourceCode, name);
  } else {
    return jsGenerateContractId(sourceCode, name);
  }
}

export async function hashContractSource(source: string): Promise<string> {
  if (WASM_AVAILABLE) {
    return await wasmHashSource(source);
  } else {
    return await jsHashSource(source);
  }
}

export function createVulnerabilityFingerprint(vuln: Vulnerability): string {
  const data = `${vuln.title}:${vuln.location.file_path}:${vuln.location.start_line}:${vuln.severity}`;
  return createHash(data);
}

export function calculateRiskScore(
  severity: Severity,
  confidence: number,
  context?: Record<string, any>
): number {
  const severityWeights: Record<Severity, number> = {
    [Severity.Critical]: 1.0,
    [Severity.High]: 0.8,
    [Severity.Medium]: 0.6,
    [Severity.Low]: 0.4,
    [Severity.Informational]: 0.2,
  };

  let baseScore = severityWeights[severity] * confidence;

  if (context) {
    baseScore *= calculateContextMultiplier(context);
  }

  return Math.min(baseScore, 1.0);
}
```

#### WASM Integration

```typescript
// WASM module interface
export interface WasmModule {
  generate_contract_id(source: string, name: string): string;
  hash_contract_source(source: string): string;
  validate_vulnerability(vuln_json: string): boolean;
  calculate_risk_score(severity: number, confidence: number): number;
}

// WASM loading and error handling
export async function loadWasmModule(): Promise<WasmModule | null> {
  try {
    // Check WebAssembly support
    if (typeof WebAssembly === 'undefined') {
      console.warn('WebAssembly not supported in this environment');
      return null;
    }

    // Dynamic import with error handling
    const wasmModule = await import('./wasm/solidity_shared_bg.wasm');

    // Initialize module
    await init(wasmModule);

    return wasmModule;
  } catch (error) {
    const wasmError = categorizeWasmError(error);
    console.warn(`WASM loading failed: ${wasmError.category}`, error);
    return null;
  }
}

// Error categorization for WASM failures
export enum WasmFailureType {
  WASM_LOADING_FAILED = 'WASM_LOADING_FAILED',
  UNSUPPORTED_BROWSER = 'UNSUPPORTED_BROWSER',
  NETWORK_ERROR = 'NETWORK_ERROR',
  INSTANTIATION_ERROR = 'INSTANTIATION_ERROR',
  UNKNOWN_ERROR = 'UNKNOWN_ERROR'
}

export interface WasmError {
  category: WasmFailureType;
  message: string;
  originalError: Error;
}

export function categorizeWasmError(error: unknown): WasmError {
  if (error instanceof Error) {
    if (error.message.includes('fetch')) {
      return {
        category: WasmFailureType.NETWORK_ERROR,
        message: 'Network error while loading WASM module',
        originalError: error
      };
    }

    if (error.message.includes('WebAssembly')) {
      return {
        category: WasmFailureType.UNSUPPORTED_BROWSER,
        message: 'Browser does not support WebAssembly',
        originalError: error
      };
    }

    return {
      category: WasmFailureType.WASM_LOADING_FAILED,
      message: error.message,
      originalError: error
    };
  }

  return {
    category: WasmFailureType.UNKNOWN_ERROR,
    message: 'Unknown WASM loading error',
    originalError: error instanceof Error ? error : new Error(String(error))
  };
}
```

## Configuration Options

### Environment Variables

```bash
# Python Configuration
SOLIDITY_SHARED_NO_RUST=1          # Disable Rust acceleration
SOLIDITY_SHARED_DEBUG=1             # Enable debug logging
SOLIDITY_SHARED_STRICT_VALIDATION=1 # Enable strict validation mode

# TypeScript Configuration (Runtime)
localStorage.setItem('SOLIDITY_SHARED_DEBUG', '1');           # Debug mode
localStorage.setItem('SOLIDITY_SHARED_NO_WASM', '1');         # Disable WASM
localStorage.setItem('SOLIDITY_SHARED_CACHE_ENABLED', '1');   # Enable caching
```

### Build Configuration

```toml
# Cargo.toml features
[features]
default = ["serde"]
python = ["pyo3"]
wasm = ["wasm-bindgen", "console_error_panic_hook"]
full = ["python", "wasm", "serde"]

# Performance features
simd = ["sha2/asm"]
parallel = ["rayon"]
```

```python
# Python setup.py configuration
rust_extensions = [
    RustExtension(
        "solidity_shared._rust",
        path="Cargo.toml",
        features=["python"],
        debug=False,  # Set to True for debug builds
    )
]
```

```javascript
// TypeScript build configuration
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "strict": true
  },
  "include": ["src/**/*", "wasm/**/*"]
}
```

## Integration Patterns

### Service Integration Pattern

```python
# Python service integration
from solidity_shared import (
    Vulnerability, Finding, Analysis,
    validate_vulnerability, generate_contract_id
)

class SecurityAnalysisService:
    def __init__(self):
        self.logger = logging.getLogger(__name__)

    async def analyze_contract(self, contract_code: str, name: str) -> Analysis:
        # Generate deterministic ID
        contract_id = generate_contract_id(contract_code, name)

        # Create analysis record
        analysis = Analysis(
            id=uuid4(),
            project_id=self.project_id,
            contract_name=name,
            contract_hash=hashlib.sha256(contract_code.encode()).hexdigest(),
            tools_used=["slither", "mythx"],
            status=AnalysisStatus.RUNNING,
            started_at=datetime.utcnow(),
            metadata={"contract_id": str(contract_id)}
        )

        return analysis
```

### Error Handling Pattern

```typescript
// TypeScript error handling with WASM fallback
export async function safeGenerateContractId(
  sourceCode: string,
  name: string
): Promise<string> {
  try {
    // Try WASM first
    if (WASM_AVAILABLE) {
      return await wasmGenerateContractId(sourceCode, name);
    }
  } catch (error) {
    console.warn('WASM operation failed, falling back to JS:', error);
  }

  // Fallback to JavaScript implementation
  return jsGenerateContractId(sourceCode, name);
}
```

### Caching Pattern

```rust
// Rust caching pattern for expensive operations
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

pub struct CachedValidator {
    cache: Arc<RwLock<HashMap<String, bool>>>,
}

impl CachedValidator {
    pub fn validate_with_cache(&self, key: &str, data: &str) -> Result<bool, ValidationError> {
        // Check cache first
        if let Some(&result) = self.cache.read().unwrap().get(key) {
            return Ok(result);
        }

        // Perform expensive validation
        let result = expensive_validation(data)?;

        // Cache result
        self.cache.write().unwrap().insert(key.to_string(), result);

        Ok(result)
    }
}
```

## Testing Specifications

### Unit Test Coverage Requirements

- **Rust Core**: >95% line coverage
- **Python Bindings**: >90% line coverage
- **TypeScript Bindings**: >90% line coverage
- **Integration Tests**: 100% of public API surface

### Performance Test Benchmarks

```rust
// Rust performance benchmarks
#[cfg(test)]
mod benchmarks {
    use super::*;
    use criterion::{criterion_group, criterion_main, Criterion};

    fn benchmark_contract_id_generation(c: &mut Criterion) {
        let source = include_str!("../fixtures/large_contract.sol");
        let name = "LargeContract";

        c.bench_function("generate_contract_id", |b| {
            b.iter(|| generate_contract_id(source, name))
        });
    }

    fn benchmark_vulnerability_validation(c: &mut Criterion) {
        let vuln = create_test_vulnerability();

        c.bench_function("validate_vulnerability", |b| {
            b.iter(|| validate_vulnerability(&vuln))
        });
    }

    criterion_group!(benches, benchmark_contract_id_generation, benchmark_vulnerability_validation);
    criterion_main!(benches);
}
```

### Cross-Language Compatibility Tests

```python
# Python-Rust compatibility test
def test_cross_language_serialization():
    # Create in Python
    vuln = Vulnerability(
        id=uuid4(),
        title="Test Vulnerability",
        severity=Severity.HIGH,
        confidence=0.95,
        # ... other fields
    )

    # Serialize to JSON
    json_data = vuln.json()

    # Deserialize in Rust via PyO3
    rust_vuln = rust_deserialize_vulnerability(json_data)

    # Verify equality
    assert rust_vuln.id == str(vuln.id)
    assert rust_vuln.title == vuln.title
    assert rust_vuln.severity == vuln.severity.value
```

This technical specification provides the detailed API documentation and integration patterns needed for developers to effectively use the `solidity-security-shared` library across all three supported languages.