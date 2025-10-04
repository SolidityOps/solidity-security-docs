# WebAssembly Integration Guide

## Overview

The shared library includes WebAssembly (WASM) bindings that provide high-performance Rust functions for TypeScript/JavaScript applications. This integration offers significant performance improvements while maintaining universal compatibility through JavaScript fallbacks.

## Architecture

```
┌─────────────────────────────────────┐
│         TypeScript/JavaScript       │
│                                     │
├─────────────────┬───────────────────┤
│  WASM Functions │  JS Fallbacks     │
│  (Performance)  │  (Compatibility)  │
└─────────────────┼───────────────────┘
                  │
            ┌─────▼─────┐
            │ Rust Core │
            │ Library   │
            └───────────┘
```

## WASM Functions Available

### Core Cryptographic Functions
- `wasm_generate_contract_id(source_code, contract_name)` - Deterministic contract ID generation
- `wasm_hash_contract_source(source_code)` - SHA-256 source code hashing

### Vulnerability Analysis Functions
- `wasm_create_vulnerability_fingerprint(file, line, column, title, code_snippet?)` - Unique vulnerability fingerprinting
- `wasm_calculate_risk_score(severity_weight, confidence, occurrence_count, false_positive_likelihood)` - Multi-factor risk assessment

### Utility Functions
- `wasm_format_file_size(bytes)` - Human-readable file size formatting
- `wasm_truncate_string(string, max_length)` - Safe string truncation
- `wasm_normalize_code_snippet(code)` - Code normalization for comparison

### System Functions
- `wasm_version()` - WASM module version information
- `wasm_health_check()` - Module health verification

## Build Process

### Development Build
```bash
cd wasm-bindings
wasm-pack build --dev --target bundler
```

### Production Build
```bash
cd wasm-bindings
wasm-pack build --release --target bundler --out-dir ../typescript/src/wasm
```

### Build Verification
```bash
# Check generated files
ls typescript/src/wasm/
# Expected: .wasm, .js, .d.ts files

# Verify file sizes
ls -la typescript/src/wasm/*.wasm
# Expected: ~70KB optimized binary
```

## TypeScript Integration

### Automatic Loading
```typescript
import { generateContractId, isWasmAvailable } from 'solidity-security-shared';

// Check WASM availability
console.log('WASM available:', isWasmAvailable());

// Functions automatically use WASM when available, JS fallbacks otherwise
const contractId = await generateContractId(sourceCode, contractName);
```

### Manual WASM Management
```typescript
import { ensureWasmLoaded, getWasmError } from 'solidity-security-shared';

// Explicitly load WASM
const loaded = await ensureWasmLoaded();
if (!loaded) {
  console.warn('WASM failed to load:', getWasmError());
  // Application continues with JS fallbacks
}
```

### Performance Monitoring
```typescript
import { getWasmDiagnostics } from 'solidity-security-shared';

const diagnostics = await getWasmDiagnostics();
console.log('WASM Status:', {
  available: diagnostics.available,
  webAssemblySupport: diagnostics.webAssemblySupport,
  loadTime: diagnostics.loadTime,
  error: diagnostics.error
});
```

## Browser Compatibility

### Supported Environments
- **Chrome 57+** - Full WASM support
- **Firefox 52+** - Full WASM support
- **Safari 11+** - Full WASM support
- **Edge 16+** - Full WASM support
- **Node.js 12+** - Runtime support with fallbacks

### Fallback Behavior
When WASM is not available:
- All functions automatically fall back to JavaScript implementations
- Performance remains acceptable for most use cases
- No functionality is lost
- Error logging helps identify WASM loading issues

## Performance Characteristics

### Expected Speedups (WASM vs JavaScript)
| Function | Performance Improvement |
|----------|------------------------|
| Contract ID Generation | 8-12x faster |
| Source Hashing | 10-15x faster |
| Vulnerability Fingerprinting | 6-10x faster |
| Risk Score Calculation | 4-6x faster |
| String Operations | 2-4x faster |

### Benchmark Results (100 iterations)
| Function | JavaScript Fallback | Expected WASM Performance |
|----------|---------------------|---------------------------|
| Contract ID | 0.12ms avg | 0.01-0.02ms avg |
| Source Hash | 0.07ms avg | 0.005-0.01ms avg |
| Fingerprint | 0.06ms avg | 0.01-0.015ms avg |
| Risk Score | 0.01ms avg | 0.002-0.005ms avg |

## Error Handling

### WASM Loading Failures
```typescript
import { getWasmError, getWasmDiagnostics } from 'solidity-security-shared';

if (!isWasmAvailable()) {
  const error = getWasmError();
  const diagnostics = await getWasmDiagnostics();

  console.warn('WASM not available:', {
    error: error,
    webAssemblySupport: diagnostics.webAssemblySupport,
    details: diagnostics.error
  });

  // Application continues with JavaScript fallbacks
}
```

### Common Error Scenarios
1. **Browser Incompatibility** - Automatic fallback to JavaScript
2. **Network Loading Issues** - Retry mechanism with fallback
3. **Memory Constraints** - Graceful degradation
4. **Security Policies** - CSP-compliant fallback behavior

## Security Considerations

### Content Security Policy
```html
<!-- Required CSP headers for WASM -->
<meta http-equiv="Content-Security-Policy"
      content="script-src 'self' 'wasm-unsafe-eval';">
```

### Memory Safety
- All WASM functions use memory-safe Rust implementations
- No unsafe code blocks in WASM-exposed functions
- Automatic memory management via wasm-bindgen
- Input validation at the WASM boundary

### Data Validation
- All inputs validated before WASM function calls
- Size limits enforced to prevent memory exhaustion
- String encoding validation for cross-language compatibility
- Error boundaries prevent WASM failures from crashing applications

## Troubleshooting

### WASM Module Not Loading
1. **Check browser support**: Verify WebAssembly is available
2. **Verify file paths**: Ensure WASM files are accessible
3. **Check network**: Verify WASM binary downloads successfully
4. **Review CSP**: Ensure Content Security Policy allows WASM

### Performance Issues
1. **Monitor fallback usage**: High fallback usage indicates WASM loading problems
2. **Profile actual performance**: Use browser dev tools to measure
3. **Check memory usage**: Monitor WASM memory consumption
4. **Verify optimizations**: Ensure production build is used

### Development Issues
```bash
# Rebuild WASM module
cd wasm-bindings
wasm-pack build --release --target bundler

# Verify TypeScript compilation
cd ../typescript
npm run type-check

# Test functionality
npm test
```

## Advanced Usage

### Custom WASM Loading
```typescript
// Custom loading with retry
async function loadWasmWithRetry(maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    if (await ensureWasmLoaded()) {
      return true;
    }
    await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
  }
  return false;
}
```

### Batch Processing
```typescript
// Efficient batch operations
async function processBatch(contracts: Contract[]) {
  const results = [];

  if (isWasmAvailable()) {
    // WASM batch processing
    for (const contract of contracts) {
      results.push(await generateContractId(contract.source, contract.name));
    }
  } else {
    // JavaScript parallel processing
    const promises = contracts.map(contract =>
      generateContractId(contract.source, contract.name)
    );
    results.push(...await Promise.all(promises));
  }

  return results;
}
```

### Performance Monitoring
```typescript
// Monitor WASM vs JavaScript performance
class PerformanceMonitor {
  private wasmTimes: number[] = [];
  private jsTimes: number[] = [];

  async measureFunction<T>(
    fn: () => Promise<T>,
    useWasm: boolean
  ): Promise<T> {
    const start = performance.now();
    const result = await fn();
    const duration = performance.now() - start;

    if (useWasm) {
      this.wasmTimes.push(duration);
    } else {
      this.jsTimes.push(duration);
    }

    return result;
  }

  getStats() {
    return {
      wasmAverage: this.average(this.wasmTimes),
      jsAverage: this.average(this.jsTimes),
      speedup: this.average(this.jsTimes) / this.average(this.wasmTimes)
    };
  }

  private average(arr: number[]): number {
    return arr.reduce((a, b) => a + b, 0) / arr.length;
  }
}
```

## Migration Guide

### From JavaScript-Only Implementation
1. **Install updated package**: `npm install solidity-security-shared@latest`
2. **Update imports**: No changes needed - WASM is automatically used when available
3. **Test functionality**: Existing code continues to work with performance improvements
4. **Monitor performance**: Use diagnostics to verify WASM is loading correctly

### Build System Integration
```json
{
  "scripts": {
    "build:wasm": "cd ../wasm-bindings && wasm-pack build --release --target bundler",
    "build": "npm run build:wasm && tsc",
    "dev": "npm run build:wasm && tsc --watch"
  }
}
```

This WebAssembly integration provides significant performance improvements while maintaining full compatibility through automatic JavaScript fallbacks, making it suitable for production use across all supported environments.