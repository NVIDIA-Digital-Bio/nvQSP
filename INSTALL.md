# Installation Guide

This software is licensed under the
[NVIDIA Software License Agreement](https://www.nvidia.com/en-us/agreements/enterprise-software/nvidia-software-license-agreement/)
and the [Product-Specific Terms for AI Products](https://www.nvidia.com/en-us/agreements/enterprise-software/product-specific-terms-for-ai-products/).
By installing this software you agree to the terms of both licenses.

## System Requirements

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Linux x86_64 | Ubuntu 20.04+ / RHEL 8+ |
| GPU | NVIDIA Ampere (sm_80) | H100 or H200 (sm_90) |
| Driver | 525+ | 535+ |
| CUDA runtime | 12.0+ | 12.4+ |
| Python | 3.8+ (for Python API) | 3.10+ |

The CUDA **Toolkit** is not required. The solver only needs the NVIDIA driver
and CUDA runtime, which are included with the driver package.

### Verifying your GPU

```bash
nvidia-smi
```

You should see your GPU model and driver version. If this command is not found,
install the NVIDIA driver first.

---

## Option 1: pip install from PyPI (recommended)

The wheel bundles the compiled solver library inside the Python package. No
system-wide installation or root access is needed.

### Install

```bash
pip install nvqsp==0.1.0
```

### Dependencies

- **numpy** (installed automatically by pip)
- **scipy** (optional, useful for building CSR matrices with `scipy.sparse`)

### Verify

```bash
python3 -c "
from nvqsp import sparse, __version__
from nvqsp.options import SparseOptions
print(f'nvQSP {__version__} loaded successfully')
print(f'Default options: rtol={SparseOptions().rtol}, atol={SparseOptions().atol}')
"
```

Expected output:

```
nvQSP 0.1.0 loaded successfully
Default options: rtol=1e-06, atol=1e-12
```

### Verify GPU execution

```python
import numpy as np
from nvqsp import sparse

A0 = np.array([0.0])
A1_rowptr = np.array([0, 1], dtype=np.int32)
A1_col    = np.array([0], dtype=np.int32)
A1_val    = np.array([-0.1])
A2_rowptr = np.array([0, 1], dtype=np.int32)
A2_col1   = np.array([0], dtype=np.int32)
A2_col2   = np.array([0], dtype=np.int32)
A2_val    = np.array([1e-30])

result = sparse.solve(
    A0=A0,
    A1_csr=(A1_rowptr, A1_col, A1_val),
    A2_csr=(A2_rowptr, A2_col1, A2_col2, A2_val),
    y0=np.array([[100.0]]),
    times=np.array([1.0, 10.0, 24.0]),
)

print(f"y(1)  = {result.y[0, 0, 0]:.4f}")   # ~90.48
print(f"y(10) = {result.y[0, 1, 0]:.4f}")   # ~36.79
print(f"y(24) = {result.y[0, 2, 0]:.4f}")   # ~9.07
print(f"Steps: {result.steps}")
```

This runs a single-compartment exponential decay (y = 100 * exp(-0.1t)) on the
GPU. If you see plausible values, the solver is working correctly.

### Virtual environments

The wheel works with virtualenv, conda, and Docker:

```bash
# virtualenv
python3 -m venv qsp_env
source qsp_env/bin/activate
pip install nvqsp==0.1.0

# conda
conda create -n qsp python=3.11 numpy -y
conda activate qsp
pip install nvqsp==0.1.0
```

### Custom library path

If you need to use a different `.so` file (e.g. a custom build), pass
`lib_path` to the solver:

```python
result = sparse.solve(..., lib_path="/path/to/libsparse_rodas4.so")
```

Or set the environment variable:

```bash
export QSP_ENGINE_LIB_SPARSE=/path/to/libsparse_rodas4.so
```

---

## Option 2: Debian Package (C/C++ users)

The `.deb` installs the shared library and public C headers system-wide.
Download `nvqsp_0.1.0_amd64.deb` from the
[GitHub release](https://github.com/NVIDIA-Digital-Bio/nvQSP/releases/tag/v0.1.0).

### Install

```bash
sudo dpkg -i nvqsp_0.1.0_amd64.deb
```

### What gets installed

| Path | Contents |
|---|---|
| `/usr/lib/nvqsp/libsparse_rodas4.so` | Solver shared library |
| `/usr/include/nvqsp/sparse_rodas4.h` | Public C API header |
| `/usr/include/nvqsp/sparse_rodas4_options.h` | Options struct definition |

### Compile and link

```bash
# C
gcc -o my_app my_app.c \
    -I/usr/include/nvqsp \
    -L/usr/lib/nvqsp -lsparse_rodas4 \
    -Wl,-rpath,/usr/lib/nvqsp -lm

# C++
g++ -o my_app my_app.cpp \
    -I/usr/include/nvqsp \
    -L/usr/lib/nvqsp -lsparse_rodas4 \
    -Wl,-rpath,/usr/lib/nvqsp
```

The `-Wl,-rpath` flag embeds the library search path in the binary so you don't
need to set `LD_LIBRARY_PATH` at runtime.

### Verify

```bash
ls -la /usr/lib/nvqsp/libsparse_rodas4.so
ls -la /usr/include/nvqsp/sparse_rodas4.h
```

### Uninstall

```bash
sudo dpkg -r nvqsp
```

---

## Option 3: Standalone Shared Library

For environments where neither pip nor dpkg is available, download
`libsparse_rodas4.so` from the
[GitHub release](https://github.com/NVIDIA-Digital-Bio/nvQSP/releases/tag/v0.1.0)
and use it directly.

### Copy to your project

```bash
cp libsparse_rodas4.so /path/to/your/project/lib/
```

### Link (C/C++)

```bash
gcc -o my_app my_app.c \
    -I/path/to/headers \
    -L/path/to/your/project/lib -lsparse_rodas4 \
    -Wl,-rpath,/path/to/your/project/lib -lm
```

### Use from Python (ctypes)

```python
import ctypes
lib = ctypes.CDLL("/path/to/libsparse_rodas4.so")
# Set up argtypes/restype manually — see API_REFERENCE.md for the full signature
```

Or install the wheel and override the library path:

```python
from nvqsp import sparse
result = sparse.solve(..., lib_path="/path/to/libsparse_rodas4.so")
```

---

## Troubleshooting

### `CUDA error: no CUDA-capable device is detected`

- Verify the GPU is visible: `nvidia-smi`
- Check that the NVIDIA driver is loaded: `lsmod | grep nvidia`
- Inside Docker, ensure `--gpus all` or `--runtime=nvidia` is passed

### `CUDA error: invalid device function`

The GPU architecture is not included in the fat binary. This release supports
sm_80, sm_89, and sm_90. Older GPUs (Volta, Turing) are not supported.

### `OSError: libcuda.so.1: cannot open shared object file`

The NVIDIA driver is not installed or not on the library path:

```bash
ldconfig -p | grep libcuda
```

If empty, install the NVIDIA driver.

### `ImportError: No module named nvqsp`

The wheel is not installed in the active Python environment:

```bash
pip list | grep nvqsp
```

### GPU out of memory

The solver allocates memory proportional to `batch_size * neq`. For large
models, reduce the batch size.
