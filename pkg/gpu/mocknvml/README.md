# Mock NVML Library

A configurable mock implementation of NVIDIA's NVML (NVIDIA Management Library) that allows testing GPU-dependent software without physical NVIDIA hardware.

## Quick Start

```bash
# Build (requires Linux with Go and GCC)
cd pkg/gpu/mocknvml
make

# Test all 3 scenarios:

# 1. Default (8x Mock A100, no config)
LD_LIBRARY_PATH=. nvidia-smi

# 2. A100 profile (8x A100-SXM4-40GB, 40GB, 400W)
LD_LIBRARY_PATH=. MOCK_NVML_CONFIG=configs/mock-nvml-config-a100.yaml nvidia-smi

# 3. GB200 profile (8x GB200 NVL, 192GB, 1000W)
LD_LIBRARY_PATH=. MOCK_NVML_CONFIG=configs/mock-nvml-config-gb200.yaml nvidia-smi
```

## Overview

This library provides a drop-in replacement for `libnvidia-ml.so` that:

- Works with the real `nvidia-smi` binary
- Supports YAML-based GPU configuration
- Simulates multiple GPU profiles (A100, GB200, etc.)
- Produces valid XML output for `nvidia-smi -x -q`

## Building

### Prerequisites

- Go 1.23+ (uses CGo)
- Linux (x86_64 or arm64)
- GCC toolchain

### Build Commands

```bash
# Build the shared library (default version 550.163.01)
make

# Build with custom library version (e.g., for GB200 driver 560.35.03)
LIB_VERSION=560.35.03 make

# Build using Docker (for cross-platform builds)
make docker-build

# Build with custom version in Docker
LIB_VERSION=560.35.03 make docker-build

# Clean build artifacts
make clean
```

### Build Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `LIB_VERSION` | Library version (appears in filename) | 550.163.01 |
| `GOLANG_VERSION` | Go version for Docker builds | 1.25.0 |

This produces:
- `libnvidia-ml.so.<version>` - The actual library
- `libnvidia-ml.so.1` - Symlink (soname)
- `libnvidia-ml.so` - Symlink (linker name)

**Note:** The `LIB_VERSION` should match the `driver_version` in your YAML config for consistency.

## Usage

### Basic Usage

```bash
# Set library path to use mock instead of real NVML
export LD_LIBRARY_PATH=/path/to/mocknvml

# Run nvidia-smi (uses mock library)
nvidia-smi
```

### With YAML Configuration

```bash
export LD_LIBRARY_PATH=/path/to/mocknvml
export MOCK_NVML_CONFIG=/path/to/config.yaml

nvidia-smi
nvidia-smi -x -q  # XML output
```

### Supported nvidia-smi Commands

| Command | Description |
|---------|-------------|
| `nvidia-smi` | Default display (GPU table) |
| `nvidia-smi -L` | List GPUs with UUIDs |
| `nvidia-smi -q` | Full query (all details) |
| `nvidia-smi -q -d MEMORY` | Memory details |
| `nvidia-smi -q -d TEMPERATURE` | Temperature details |
| `nvidia-smi -q -d POWER` | Power details |
| `nvidia-smi -q -d CLOCK` | Clock details |
| `nvidia-smi -q -d ECC` | ECC details |
| `nvidia-smi -q -d UTILIZATION` | Utilization details |
| `nvidia-smi -q -d PCIE` | PCIe details |
| `nvidia-smi -x -q` | XML output (full query) |
| `nvidia-smi --query-gpu=... --format=csv` | CSV output |
| `nvidia-smi -i <index>` | Query specific GPU |

Example CSV query:

```bash
nvidia-smi --query-gpu=index,name,uuid,memory.total,power.draw,temperature.gpu --format=csv
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `MOCK_NVML_CONFIG` | Path to YAML configuration file | (none - uses defaults) |
| `MOCK_NVML_NUM_DEVICES` | Number of GPUs to simulate (if no YAML) | 8 |
| `MOCK_NVML_DRIVER_VERSION` | Driver version string (if no YAML) | 550.163.01 |
| `MOCK_NVML_DEBUG` | Enable debug logging to stderr | (disabled) |

### Debugging

Debug output is disabled by default for clean output. Enable verbose logging to troubleshoot issues:

```bash
# Enable debug logging
LD_LIBRARY_PATH=. MOCK_NVML_DEBUG=1 nvidia-smi

# Debug output shows NVML function calls, device creation, config loading, etc.
# Example output:
# [CONFIG] Loaded YAML config: 8 devices, driver 550.163.01
# [ENGINE] Creating devices from YAML config
# [DEVICE 0] Created: name=NVIDIA A100-SXM4-40GB uuid=GPU-12345678-...
# [NVML] nvmlDeviceGetHandleByIndex(0)
# [NVML] nvmlDeviceGetTemperature(sensor=0) -> 33
```

### YAML Configuration

YAML configs allow full control over GPU properties. See `configs/` for examples:

- `mock-nvml-config-a100.yaml` - DGX A100 (8x A100-SXM4-40GB)
- `mock-nvml-config-gb200.yaml` - GB200 NVL (8x GB200 with 192GB HBM3e)

#### Configuration Structure

```yaml
version: "1.0"

system:
  driver_version: "550.163.01"
  nvml_version: "12.550.163.01"
  cuda_version: "12.4"
  cuda_version_major: 12
  cuda_version_minor: 4

device_defaults:
  name: "NVIDIA A100-SXM4-40GB"
  architecture: "ampere"
  memory:
    total_bytes: 42949672960      # 40 GiB
  power:
    default_limit_mw: 400000      # 400W
    current_draw_mw: 72000        # 72W idle
  thermal:
    temperature_gpu_c: 33
  # ... see full examples in configs/

devices:
  - index: 0
    uuid: "GPU-12345678-1234-1234-1234-123456780000"
    pci:
      bus_id: "00000000:07:00.0"
  - index: 1
    uuid: "GPU-12345678-1234-1234-1234-123456780001"
    pci:
      bus_id: "00000000:0F:00.0"
  # ... define each GPU
```

## Example Output

### Default (No Config)

```
$ LD_LIBRARY_PATH=. nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.01             Driver Version: 550.163.01     CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|=========================================+========================+======================|
|   0  Mock NVIDIA A100-SXM4-40GB     Off |   0000:00:00.0     Off |                  Off |
|  0%   N/A    P0             N/A /  N/A  |       0MiB /  40960MiB |      0%      Default |
...
```

### With A100 Config

```
$ MOCK_NVML_CONFIG=configs/mock-nvml-config-a100.yaml LD_LIBRARY_PATH=. nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.01             Driver Version: 550.163.01     CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|=========================================+========================+======================|
|   0  NVIDIA A100-SXM4-40GB          On  |   00000000:07:00.0 Off |                    0 |
| N/A   33C    P0             72W /  400W |       0MiB /  40960MiB |      0%      Default |
...
```

### With GB200 Config

```
$ MOCK_NVML_CONFIG=configs/mock-nvml-config-gb200.yaml LD_LIBRARY_PATH=. nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.163.01             Driver Version: 560.35.03      CUDA Version: 12.6     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|=========================================+========================+======================|
|   0  NVIDIA GB200 NVL               On  |   00000000:0A:00.0 Off |                    0 |
| N/A   36C    P0            145W / 1000W |       0MiB / 196608MiB |      0%      Default |
...
```

## Architecture

```
pkg/gpu/mocknvml/
├── bridge/
│   └── bridge_generated.go    # CGo bridge (generated)
├── engine/
│   ├── config.go              # Configuration loading
│   ├── config_types.go        # YAML struct definitions
│   ├── device.go              # ConfigurableDevice implementation
│   ├── engine.go              # Main engine singleton
│   └── handles.go             # C-compatible handle management
├── configs/
│   ├── mock-nvml-config-a100.yaml
│   └── mock-nvml-config-gb200.yaml
├── Makefile
└── README.md
```

## Regenerating the Bridge

The bridge code is generated from go-nvml types:

```bash
cd /path/to/k8s-test-infra
go run ./cmd/generate-bridge
```

## Supported NVML Functions

The mock library implements 50+ NVML functions required by nvidia-smi, including:

- Device enumeration (`nvmlDeviceGetCount`, `nvmlDeviceGetHandleByIndex`)
- Device properties (`nvmlDeviceGetName`, `nvmlDeviceGetUUID`, `nvmlDeviceGetMemoryInfo`)
- Thermal/Power (`nvmlDeviceGetTemperature`, `nvmlDeviceGetPowerUsage`)
- Clocks (`nvmlDeviceGetClockInfo`, `nvmlDeviceGetMaxClockInfo`)
- ECC (`nvmlDeviceGetEccMode`, `nvmlDeviceGetTotalEccErrors`)
- PCIe (`nvmlDeviceGetPciInfo`, `nvmlDeviceGetCurrPcieLinkGeneration`)
- MIG (`nvmlDeviceGetMigMode`)
- Events (`nvmlEventSetCreate`, `nvmlEventSetWait`)

## Limitations

- Read-only simulation (no actual GPU operations)
- Some advanced features return NOT_SUPPORTED
- No actual CUDA compute capability
- Process list is always empty

## License

Apache License 2.0 - See LICENSE file in repository root.
