# nvQSP v0.1.0

GPU-accelerated RODAS4 stiff ODE solver for Quantitative Systems Pharmacology
(QSP) and PBPK population studies.

## Distribution

| Channel | Install |
|---|---|
| **PyPI** | `pip install nvqsp==0.1.0` |
| **GitHub Release** | [nvqsp_0.1.0_amd64.deb](https://github.com/NVIDIA-Digital-Bio/nvQSP/releases/tag/v0.1.0) (C/C++ headers + lib) |
| **GitHub Release** | [libsparse_rodas4.so](https://github.com/NVIDIA-Digital-Bio/nvQSP/releases/tag/v0.1.0) (standalone shared library) |

All binaries are fat binaries with native code for:
- **sm_80** — Ampere (A100, A10)
- **sm_89** — Ada Lovelace (L4, L40, RTX 4090)
- **sm_90** — Hopper (H100, H200)
- **compute_90 PTX** — forward compatibility for future architectures (Blackwell, etc.)

## Requirements

- Linux x86_64
- NVIDIA GPU: Ampere (sm_80), Ada Lovelace (sm_89), or Hopper (sm_90)
- NVIDIA driver 525+ (CUDA runtime 12.0+)
- Python 3.8+ with NumPy (for the Python API)

**No CUDA Toolkit required** to run the solver. The toolkit is only needed to
build from source.

## Quick Install

**Python (from PyPI):**
```bash
pip install nvqsp==0.1.0
```

**C/C++ (Debian/Ubuntu):**

Download `nvqsp_0.1.0_amd64.deb` from the
[GitHub release](https://github.com/NVIDIA-Digital-Bio/nvQSP/releases/tag/v0.1.0), then:
```bash
sudo dpkg -i nvqsp_0.1.0_amd64.deb
```

See [INSTALL.md](INSTALL.md) for full details.

## Quick Start

```python
import numpy as np
from scipy.sparse import csr_matrix
from nvqsp import sparse
from nvqsp.options import SparseOptions

# Two-compartment model: dy/dt = A0 + A1*y + A2*(y x y)
neq = 2
A0 = np.array([0.0, 0.0])

A1 = csr_matrix([[-0.3, 0.1], [0.3, -0.1]])
A1_rowptr = A1.indptr.astype(np.int32)
A1_col    = A1.indices.astype(np.int32)
A1_val    = A1.data.astype(np.float64)

# A2 must have >= 1 entry; use epsilon for purely linear models
A2_rowptr = np.array([0, 1, 1], dtype=np.int32)
A2_col1   = np.array([0], dtype=np.int32)
A2_col2   = np.array([0], dtype=np.int32)
A2_val    = np.array([1e-30])

# 100 patients, 48 time points, dose of 100 mg at t=0
result = sparse.solve(
    A0=A0,
    A1_csr=(A1_rowptr, A1_col, A1_val),
    A2_csr=(A2_rowptr, A2_col1, A2_col2, A2_val),
    y0=np.tile([10.0, 0.0], (100, 1)),
    times=np.linspace(1.0, 24.0, 48),
    doses=[(0.0, 100.0)],
    opts=SparseOptions(rtol=1e-6, atol=1e-9),
)

print(result.y.shape)   # (100, 48, 2)
print(result.steps)     # total ODE steps across all patients
```

See [API_REFERENCE.md](API_REFERENCE.md) for the complete Python and C API.

## Model Form

The solver handles polynomial ODE systems of the form:

```
dy/dt = A0 + A1 * y + A2 * (y x y)
```

| Term | Shape | Meaning |
|---|---|---|
| A0 | (neq,) | Zeroth-order: constant synthesis, zero-order infusion |
| A1 | (neq, neq) sparse CSR | First-order: linear elimination, transfer rates |
| A2 | (neq, neq, neq) sparse CSR | Second-order: bilinear / mass-action terms |

**Covers:** all linear PBPK models, first-order absorption, IV bolus/infusion,
bimolecular mass-action kinetics (drug-receptor binding, target-mediated
disposition with second-order approximation).

**Does not cover:** Michaelis-Menten elimination, Hill-function PD, TMDD with
quasi-steady-state, indirect response models, DAE systems.

## Documentation

- [INSTALL.md](INSTALL.md) — Installation and verification
- [API_REFERENCE.md](API_REFERENCE.md) — Complete Python and C API reference
- [CHANGELOG.md](CHANGELOG.md) — Release notes and known limitations

## License

This software is licensed under the
[NVIDIA Software License Agreement](https://www.nvidia.com/en-us/agreements/enterprise-software/nvidia-software-license-agreement/)
and the [Product-Specific Terms for AI Products](https://www.nvidia.com/en-us/agreements/enterprise-software/product-specific-terms-for-ai-products/).
By downloading, installing, or using this software you agree to the terms of
both licenses.
