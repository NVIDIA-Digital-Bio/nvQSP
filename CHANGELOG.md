# Changelog

## v0.1.0 — 2026-03-09

Initial public release.

### Solver

- RODAS4 (4th-order Rosenbrock) stiff ODE solver for polynomial systems of the
  form `dy/dt = A0 + A1*y + A2*(y x y)`.
- Jacobi-preconditioned BiCGSTAB iterative linear solver with configurable
  tolerances and iteration limits.
- Adaptive step-size control with error estimation and automatic
  accept/reject logic.
- Two GPU backends (automatically selected at runtime):
  - **Single-kernel**: persistent block-per-patient, shared-memory resident.
    Best for models up to ~2,800 states.
  - **Multi-kernel**: host-driven, global-memory based. Handles arbitrarily
    large models.
- Per-patient parameter variation for A0, A1, and A2 coefficient values with
  shared sparsity structure.
- Discrete dosing events applied to state index 0. Dose schedule (times and
  amounts) is shared across all patients in the batch.
- Bit-exact reproducibility across runs on the same GPU.

### Python API

- `nvqsp.sparse.solve()` — high-level interface with automatic batch size
  inference, flexible input shapes, and library auto-discovery.
- `nvqsp.options.SparseOptions` — dataclass with all solver parameters and
  sensible defaults.

### C API

- `sparse_rodas4_solve()` — single function entry point with CSR matrix inputs,
  batch support, and optional dosing.
- `SparseRodas4Options` typedef — C-compatible struct (works with both C and
  C++ compilers).

### Packaging

- pip wheel (`nvqsp-0.1.0-py3-none-linux_x86_64.whl`) with bundled `.so`.
- Debian package (`nvqsp_0.1.0_amd64.deb`) with system-wide library and headers.
- Fat binary with native SASS for sm_80 (Ampere), sm_89 (Ada Lovelace),
  sm_90 (Hopper), and compute_90 PTX for forward compatibility.

### Known Limitations

- Dosing is hardcoded to state index 0. Models with a different dosing
  compartment must reorder the state vector before calling the solver.
- `A2_nnz` must be >= 1. For purely linear models, add a single entry with
  value `1e-30`.
- CUDA errors call `exit(1)`. The solver does not return error codes; GPU
  failures terminate the process.
- Does not support Michaelis-Menten elimination, Hill-function PD, TMDD,
  indirect response models, or DAE systems.
- Dose schedule (both times and amounts) is shared across all patients in the
  batch. Per-patient dosing is not supported; however, per-patient PK variation
  can be achieved through per-patient A0, A1, and A2 coefficient values.
