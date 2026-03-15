# API Reference

## Python API

### Module: `nvqsp.sparse`

#### `sparse.solve()`

Solves a batch of stiff ODE systems on the GPU using the RODAS4 method.

```python
from nvqsp import sparse

result = sparse.solve(
    A0,
    A1_csr,
    A2_csr,
    y0,
    times,
    doses=None,
    opts=None,
    lib_path=None,
)
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `A0` | `ndarray (neq,)` or `(batch, neq)` | Zeroth-order coefficients. Shape `(neq,)` is broadcast to all patients. Shape `(batch, neq)` provides per-patient values. |
| `A1_csr` | `tuple (rowptr, col, val)` | First-order coefficient matrix in CSR format. `rowptr`: `int32 (neq+1,)`. `col`: `int32 (A1_nnz,)`. `val`: `float64 (A1_nnz,)` or `(batch, A1_nnz)`. |
| `A2_csr` | `tuple (rowptr, col1, col2, val)` | Second-order bilinear coefficient tensor in CSR format. `rowptr`: `int32 (neq+1,)`. `col1`, `col2`: `int32 (A2_nnz,)`. `val`: `float64 (A2_nnz,)` or `(batch, A2_nnz)`. |
| `y0` | `ndarray (neq,)` or `(batch, neq)` | Initial conditions. Shape `(neq,)` is broadcast to all patients. |
| `times` | `ndarray (n_times,)` | Output time points. Must be strictly increasing and positive. |
| `doses` | `list of (float, float)` or `None` | Dosing events as `(time, amount)` pairs. Dose is added to state index 0. `None` or empty list means no dosing. |
| `opts` | `SparseOptions` or `None` | Solver options. `None` uses library defaults. |
| `lib_path` | `str`, `Path`, or `None` | Override path to `libsparse_rodas4.so`. `None` uses the bundled library. |

**Returns:** `SolveResult`

| Field | Type | Description |
|---|---|---|
| `y` | `ndarray (batch, n_times, neq)` | Solution trajectories. `y[b, t, i]` is state `i` of patient `b` at time `times[t]`. |
| `steps` | `int` | Total ODE steps taken across all patients. |

**Raises:**

| Exception | Condition |
|---|---|
| `FileNotFoundError` | Library `.so` not found at any search location. |
| `ValueError` | `A2_nnz == 0` (at least one A2 entry is required). |

**Notes:**

- Batch size is inferred from `A0` or `y0` if they are 2-D. If both are 1-D,
  batch size defaults to 1.
- CSR value arrays accept three shapes: `(nnz,)` broadcasts to all patients,
  `(batch, nnz)` provides per-patient values, `(batch * nnz,)` is treated as
  pre-flattened per-patient values.
- The library is loaded once and cached for subsequent calls.

---

### Module: `nvqsp.options`

#### `SparseOptions`

Dataclass controlling solver behaviour. All fields have sensible defaults.

```python
from nvqsp.options import SparseOptions

opts = SparseOptions(
    rtol             = 1e-6,
    atol             = 1e-12,
    fac_min          = 0.2,
    fac_max          = 10.0,
    fac_safe         = 0.9,
    fac_rej          = 0.1,
    max_steps        = 20000,
    linsol_rtol      = 1e-8,
    linsol_atol      = 1e-12,
    linsol_max_iters = 30,
)
```

**Fields:**

| Field | Type | Default | Description |
|---|---|---|---|
| `rtol` | float | 1e-6 | Relative tolerance for the ODE solution. Controls the accepted error relative to the solution magnitude. |
| `atol` | float | 1e-12 | Absolute tolerance for the ODE solution. Dominates for state values near zero. |
| `fac_min` | float | 0.2 | Minimum step-size growth/shrink factor. Prevents the step size from changing by more than 5x in one step. |
| `fac_max` | float | 10.0 | Maximum step-size growth factor. Limits aggressive step-size increases after easy steps. |
| `fac_safe` | float | 0.9 | Safety factor applied to the optimal step-size estimate. Values < 1.0 make the controller conservative. |
| `fac_rej` | float | 0.1 | Factor applied after a rejected step. On rejection, `h_new = h * max(fac_rej, computed_factor)`. |
| `max_steps` | int | 20000 | Maximum ODE steps per integration interval (between dose events). If exceeded, the solver returns the last accepted state. |
| `linsol_rtol` | float | 1e-8 | Relative tolerance for the BiCGSTAB iterative linear solver used in each RODAS4 stage. |
| `linsol_atol` | float | 1e-12 | Absolute tolerance for the BiCGSTAB solver. |
| `linsol_max_iters` | int | 30 | Maximum BiCGSTAB iterations per linear solve. |

**Guidance on tuning:**

- For most QSP/PBPK models, the defaults work well.
- If the solver takes too many steps, try relaxing `rtol` (e.g. `1e-4`).
- If solutions show oscillation or instability, tighten `linsol_rtol` or
  increase `linsol_max_iters`.
- For very stiff systems (stiffness ratio > 10^8), increasing `max_steps` to
  50,000 may be needed.
- Passing `opts=None` to `sparse.solve()` uses the same defaults listed above.

---

### Library search order

When `lib_path` is not specified, the Python wrapper searches for the shared
library in this order:

1. **Bundled:** `<nvqsp_package>/lib/libsparse_rodas4.so` (inside the wheel)
2. **Environment variable:** `$QSP_ENGINE_LIB_SPARSE`
3. **System search:** `ctypes.util.find_library("sparse_rodas4")`

---

## Dosing

Doses are specified as a list of `(time, amount)` tuples. Each dose adds
`amount` to **state index 0** (the primary dosing compartment).

```python
doses = [
    (0.0,  100.0),    # 100 mg at t=0
    (24.0,  50.0),    # 50 mg at t=24 h
    (48.0,  50.0),    # 50 mg at t=48 h
]
```

**Conventions:**

| Behaviour | Detail |
|---|---|
| Dose at t=0 | Applied to `y0` **before** integration starts (within tolerance 1e-8). |
| Mid-run doses | Solver integrates to the dose time, applies the dose, then continues. |
| Output at dose time | Recorded **post-dose** (after the dose is applied). |
| Dose target | Always state index 0. Reorder the state vector if your dosing compartment differs. |
| Dose schedule | Both times and amounts are shared across all patients in the batch. Per-patient PK variation is supported through A0/A1/A2 coefficient values, not through dosing. |
| No doses | Pass `doses=None` or `doses=[]`. |

---

## Per-Patient Parameter Variation

All three coefficient arrays support independent values per patient. The
sparsity structure (row pointers and column indices) is shared — only the
**values** differ.

| Array | Shared shape | Per-patient shape |
|---|---|---|
| `A0` | `(neq,)` | `(batch, neq)` |
| `A1_csr val` | `(A1_nnz,)` | `(batch, A1_nnz)` |
| `A2_csr val` | `(A2_nnz,)` | `(batch, A2_nnz)` |

**Example: per-patient clearance variation**

```python
import numpy as np
from nvqsp import sparse

batch_size = 1000
A1_nnz = len(A1_col)

# Lognormal clearance distribution
cl = np.random.lognormal(mean=np.log(0.5), sigma=0.2, size=batch_size)

# Build per-patient A1 values
A1_val_batch = np.tile(A1_val_base, (batch_size, 1))   # (batch, A1_nnz)
A1_val_batch[:, cl_index] = -cl                         # vary clearance

result = sparse.solve(
    A0=A0,
    A1_csr=(A1_rowptr, A1_col, A1_val_batch),
    A2_csr=(A2_rowptr, A2_col1, A2_col2, A2_val),
    y0=y0,
    times=times,
)
# result.y[i] is the trajectory for patient i
```

Per-patient variation produces bit-exact results compared to singleton runs.

---

## C API

### Header: `sparse_rodas4.h`

```c
#include "sparse_rodas4.h"
```

### `sparse_rodas4_solve()`

```c
void sparse_rodas4_solve(
    int batch_size,
    int neq,
    const double* h_A0,              /* [batch_size * neq]              */
    int A1_nnz,
    const int*    h_A1_rowptr,       /* [neq + 1]                      */
    const int*    h_A1_col,          /* [A1_nnz]                       */
    const double* h_A1_val,          /* [batch_size * A1_nnz]           */
    int A2_nnz,
    const int*    h_A2_rowptr,       /* [neq + 1]                      */
    const int*    h_A2_col1,         /* [A2_nnz]                       */
    const int*    h_A2_col2,         /* [A2_nnz]                       */
    const double* h_A2_val,          /* [batch_size * A2_nnz]           */
    const double* h_times,           /* [n_times]                      */
    int n_times,
    const double* h_y0,              /* [batch_size * neq]              */
    const double* h_dose_times,      /* [n_doses]                      */
    const double* h_dose_amounts,    /* [n_doses]                      */
    int n_doses,
    const SparseRodas4Options* opts, /* NULL for defaults               */
    double* h_y_out,                 /* [batch_size * n_times * neq]    */
    int* h_total_steps               /* out, may be NULL                */
);
```

**Parameters:**

| Parameter | Description |
|---|---|
| `batch_size` | Number of patients to solve in parallel. |
| `neq` | Number of ODE states per patient. |
| `h_A0` | Zeroth-order coefficients. Row-major: patient b starts at `b * neq`. |
| `A1_nnz` | Number of nonzero entries in the A1 matrix. |
| `h_A1_rowptr` | CSR row pointers for A1. Shared across all patients. |
| `h_A1_col` | CSR column indices for A1. Shared across all patients. |
| `h_A1_val` | CSR values for A1. Patient b starts at `b * A1_nnz`. |
| `A2_nnz` | Number of nonzero entries in the A2 tensor. Must be >= 1. |
| `h_A2_rowptr` | CSR row pointers for A2. Shared across all patients. |
| `h_A2_col1` | First column indices for A2. `A2[i, col1[p], col2[p]]` is the p-th entry in row i. |
| `h_A2_col2` | Second column indices for A2. |
| `h_A2_val` | Values for A2. Patient b starts at `b * A2_nnz`. |
| `h_times` | Strictly increasing output time points. |
| `n_times` | Number of output time points. |
| `h_y0` | Initial conditions. Patient b starts at `b * neq`. |
| `h_dose_times` | Dose event times. NULL if n_doses is 0. |
| `h_dose_amounts` | Dose amounts. NULL if n_doses is 0. |
| `n_doses` | Number of dose events. 0 for no dosing. |
| `opts` | Solver options. Pass NULL to use library defaults. |
| `h_y_out` | Output buffer. Patient b, time t, state i is at `b * n_times * neq + t * neq + i`. |
| `h_total_steps` | Receives the total ODE steps taken. May be NULL. |

### `SparseRodas4Options`

```c
#include "sparse_rodas4_options.h"

typedef struct SparseRodas4Options {
    double rtol;            /* 1e-6   — relative ODE tolerance           */
    double atol;            /* 1e-12  — absolute ODE tolerance           */
    double facMin;          /* 0.2    — minimum step-size factor         */
    double facMax;          /* 10.0   — maximum step-size factor         */
    double facSafe;         /* 0.9    — safety factor                    */
    double facRej;          /* 0.1    — factor after step rejection      */
    int    max_steps;       /* 20000  — max ODE steps per interval       */
    double linsol_rtol;     /* 1e-8   — BiCGSTAB relative tolerance      */
    double linsol_atol;     /* 1e-12  — BiCGSTAB absolute tolerance      */
    int    linsol_max_iters;/* 30     — BiCGSTAB iteration limit         */
} SparseRodas4Options;
```

The struct is 80 bytes (includes internal padding after `max_steps`). Pass NULL
instead of an options pointer to use library defaults.

### Array memory layout

All arrays are row-major (C order), contiguous in memory.

| Array | Layout | Stride |
|---|---|---|
| `h_A0` | `[patient_0 \| patient_1 \| ...]` | `neq` |
| `h_A1_val` | `[patient_0_vals \| patient_1_vals \| ...]` | `A1_nnz` |
| `h_A2_val` | `[patient_0_vals \| patient_1_vals \| ...]` | `A2_nnz` |
| `h_y0` | `[patient_0_ic \| patient_1_ic \| ...]` | `neq` |
| `h_y_out` | `[patient_0_traj \| patient_1_traj \| ...]` | `n_times * neq` |

Within each patient's trajectory, the inner layout is:
`[time_0: state_0..state_N] [time_1: state_0..state_N] ...`

### Compile and link example

```c
#include <stdio.h>
#include <math.h>
#include "sparse_rodas4.h"

int main() {
    int neq = 1, batch = 1, n_times = 3, n_doses = 1;

    double A0[] = {0.0};
    int    A1_rowptr[] = {0, 1};
    int    A1_col[]    = {0};
    double A1_val[]    = {-0.1};
    int    A2_rowptr[] = {0, 1};
    int    A2_col1[]   = {0};
    int    A2_col2[]   = {0};
    double A2_val[]    = {1e-30};

    double times[]        = {1.0, 10.0, 24.0};
    double y0[]           = {0.0};
    double dose_times[]   = {0.0};
    double dose_amounts[] = {100.0};

    double y_out[3];
    int steps;

    sparse_rodas4_solve(
        batch, neq, A0,
        1, A1_rowptr, A1_col, A1_val,
        1, A2_rowptr, A2_col1, A2_col2, A2_val,
        times, n_times, y0,
        dose_times, dose_amounts, n_doses,
        NULL,       /* use defaults */
        y_out, &steps
    );

    for (int t = 0; t < n_times; t++)
        printf("y(%.0f) = %.4f  (expected %.4f)\n",
               times[t], y_out[t], 100.0 * exp(-0.1 * times[t]));
    printf("Steps: %d\n", steps);

    return 0;
}
```

Compile:

```bash
gcc -o test_api test_api.c \
    -I/usr/include/nvqsp \
    -L/usr/lib/nvqsp -lsparse_rodas4 \
    -Wl,-rpath,/usr/lib/nvqsp -lm
```

### Error handling

CUDA errors call `exit(1)` and print a diagnostic message to stderr. GPU
out-of-memory will terminate the process. Validate batch sizes against available
GPU memory before calling the solver.

---

## Backend Selection

The solver automatically selects between two GPU kernel backends at runtime:

| Backend | When selected | Strategy |
|---|---|---|
| **Single-kernel** | `neq` fits in shared memory | One persistent CUDA block per patient. All RODAS4 stages execute in a single kernel launch. Lowest overhead. |
| **Multi-kernel** | `neq` exceeds shared memory capacity | Host-driven loop. Each RODAS4 stage is a separate kernel launch. Uses global memory. |

Backend selection is automatic and transparent. Both backends produce identical
results.
