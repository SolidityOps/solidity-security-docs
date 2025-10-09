# Scanner Docker Images

This document describes the production-ready Docker images for security scanner tools used by the Kubernetes Jobs-based scanner execution architecture.

## Architecture Overview

The scanner images solve dependency conflicts by running each scanner in an isolated container with its own dependencies. This resolves issues like:
- **Mythril**: requires `ckzg<2`
- **Slither**: requires `ckzg>=2.0.0` (via web3>=7.10)

By running scanners as Kubernetes Jobs, each scanner gets:
- Isolated environment with specific dependencies
- Resource limits (CPU/memory)
- Automatic retry on failure
- TTL-based cleanup after completion

## Scanner Images

### 1. Aderyn (scanner-aderyn:0.1.0)
**Location**: `/Users/pwner/Git/ABS/solidity-security-tool-integration/scanner-images/aderyn/Dockerfile`

Rust-based static analyzer for Solidity smart contracts from Cyfrin.

**Build Command**:
```bash
cd /Users/pwner/Git/ABS/solidity-security-tool-integration/scanner-images/aderyn
eval $(minikube docker-env)  # For local Minikube
docker build -t scanner-aderyn:0.1.0 .
```

**Features**:
- Multi-stage build (builder + runtime)
- Clones from GitHub: https://github.com/Cyfrin/aderyn
- Compiles from source using Cargo
- Optimized runtime image (debian:bookworm-slim)
- Build time: ~4-5 minutes (Rust compilation)

**Resource Requirements**:
- Memory limit: 512Mi
- Memory request: 256Mi
- CPU limit: 1000m
- CPU request: 250m

### 2. Mythril (mythril/myth:latest)
**Public Image**: Available from Docker Hub

Symbolic execution-based security analyzer.

**Pull Command**:
```bash
docker pull mythril/myth:latest
```

**Resource Requirements**:
- Memory limit: 2Gi (symbolic execution is memory-intensive)
- Memory request: 1Gi
- CPU limit: 1000m
- CPU request: 250m

### 3. Slither (trailofbits/eth-security-toolbox:latest)
**Public Image**: Available from Docker Hub

Static analysis framework from Trail of Bits.

**Pull Command**:
```bash
docker pull trailofbits/eth-security-toolbox:latest
```

**Resource Requirements**:
- Memory limit: 1Gi
- Memory request: 512Mi
- CPU limit: 1000m
- CPU request: 250m

## Usage with KubernetesJobManager

The `KubernetesJobManager` in `solidity-security-tool-integration` uses these images:

```python
from scanners import KubernetesJobManager

job_manager = KubernetesJobManager(namespace="solidity-security")

# Create a scanner Job
job_name = job_manager.create_scanner_job(
    scan_id="uuid-scan-id",
    scanner_name="aderyn",  # or "mythril", "slither"
    contract_path="MyContract.sol"
)

# Wait for completion
status = job_manager.wait_for_job_completion(job_name, timeout=600)

# Get logs
logs = job_manager.get_job_logs(job_name)
```

## Production Deployment

### Building for Production

For production, push images to a container registry:

```bash
# Tag for your registry
docker tag scanner-aderyn:0.1.0 your-registry.com/scanner-aderyn:0.1.0

# Push to registry
docker push your-registry.com/scanner-aderyn:0.1.0
```

### Update KubernetesJobManager

Update the `_get_scanner_image()` method in `kubernetes_job_manager.py`:

```python
def _get_scanner_image(self, scanner: str) -> str:
    """Get Docker image for scanner."""
    images = {
        "mythril": "mythril/myth:latest",
        "slither": "trailofbits/eth-security-toolbox:latest",
        "aderyn": "your-registry.com/scanner-aderyn:0.1.0"
    }
    return images.get(scanner, "unknown-scanner:latest")
```

## Image Locations

**Custom Scanner Images**:
```
/Users/pwner/Git/ABS/solidity-security-tool-integration/scanner-images/
└── aderyn/
    └── Dockerfile                     # Aderyn scanner image
```

**Service Images**: Each microservice has its own Dockerfile in its repository:
- `/Users/pwner/Git/ABS/solidity-security-api-service/Dockerfile`
- `/Users/pwner/Git/ABS/solidity-security-contract-parser/Dockerfile`
- `/Users/pwner/Git/ABS/solidity-security-data-service/Dockerfile`
- `/Users/pwner/Git/ABS/solidity-security-intelligence-engine/Dockerfile`
- `/Users/pwner/Git/ABS/solidity-security-notification/Dockerfile`
- `/Users/pwner/Git/ABS/solidity-security-tool-integration/Dockerfile`
- `/Users/pwner/Git/ABS/solidity-security-dashboard/Dockerfile`

## Testing

Test scanner images with the provided test script:

```bash
cd /Users/pwner/Git/ABS/solidity-security-tool-integration
python3 test_job_manager.py
```

This validates:
- ✅ KubernetesJobManager can be initialized
- ✅ Scanner Jobs can be created
- ✅ Job status can be retrieved
- ✅ Jobs can be listed
- ✅ Jobs can be deleted

## References

- **Aderyn**: https://github.com/Cyfrin/aderyn
- **Mythril**: https://github.com/ConsenSys/mythril
- **Slither**: https://github.com/crytic/slither
- **KubernetesJobManager**: `/Users/pwner/Git/ABS/solidity-security-tool-integration/src/scanners/kubernetes_job_manager.py`
