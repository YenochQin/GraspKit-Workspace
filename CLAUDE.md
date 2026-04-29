# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Layout

This directory is a workspace holding three paired repositories that are developed together. Each subproject has its own `CLAUDE.md` with detailed commands and architecture — read it before working inside that subproject.

- **`GraspKit/`** — the **internal development library** (`grasp-kit`, currently `3.2.dev2`). This is where developers write and modify the core APIs: GRASP2018 output processing, CSF processing, ML-driven CSF selection, and plotting. **Not published as a standalone product** — it is consumed only as an editable dependency of GraspKit-Tools. See `GraspKit/CLAUDE.md` and `GraspKit/AGENTS.md`.
  - Source is split across four `src/` packages built into one wheel: `graspkit/` (main: `data_IO`, `CSFs_processor`, `grasp_data_extractor`, `ml_module`, `utils`), `graspkit_config/` (Pydantic config models), `graspkit_ml/`, `graspkit_plot/`.
- **`rCSFs/`** — the **internal Rust/Python extension** providing high-throughput CSF→Parquet conversion and descriptor generation. Compiled module name `_rcsfs`, public Python package `rcsfs`. Built with maturin/PyO3, Rust edition 2024, Python 3.14 only. Like GraspKit, **not published standalone** — consumed by GraspKit-Tools via the prebuilt wheels under `GraspKit-Tools/package/rcsfs/`. See `rCSFs/CLAUDE.md` and `rCSFs/AGENTS.md`.
- **`GraspKit-Tools/`** — the **end-user / publicly released product**. Bundles pipeline scripts, configuration UIs, and SLURM orchestration on top of `grasp-kit` and `rcsfs`. This is what external users install and run; treat its CLI/config surface as the public contract. See `GraspKit-Tools/CLAUDE.md` and `GraspKit-Tools/AGENTS.md`.
  - Entry points: `ml_CSFs_selection_scripts/` (training, config loader, run scripts, Streamlit config app under `initialization_tools/config_app/`), `pyscript/` (analysis/plotting), `scripts/` (shell helpers), and `package/rcsfs/` (prebuilt `rcsfs` wheels for Linux x86_64 and macOS arm64, cp314).

## Python Environment

**Always activate from `GraspKit-Tools/.venv`, never from `GraspKit/.venv`.** Tools' uv environment installs `grasp-kit` editable from `../graspkit`, so it provides everything Kit's own venv would plus the pipeline dependencies (`rcsfs`, Streamlit, etc.). Running scripts under Kit's venv will miss `rcsfs` and the Tools-only packages.

```bash
cd GraspKit-Tools
uv sync                             # default CPU/runtime/dev/UI/docs environment
source .venv/bin/activate           # or: uv run <cmd>
```

On CUDA hosts that need GPU PyTorch wheels, use `uv sync --no-group cpu --extra gpu`.

Edits inside `GraspKit/src/...` are picked up immediately by this environment because of the editable install — no reinstall needed.

## Cross-Repo Coupling

`GraspKit-Tools/pyproject.toml` declares `grasp-kit` as an **editable local dependency** at `../graspkit` (lowercase) via `[tool.uv.sources]`, and pulls `rcsfs` wheels from `package/rcsfs/` via `[tool.uv] find-links`. Implications:

- The repos must sit side-by-side under this workspace directory, and the Tools repo expects the Kit repo at `../graspkit`. The actual directory is named `GraspKit` (CamelCase) — this resolves on case-insensitive filesystems (default macOS APFS) but **will break on case-sensitive Linux**. If running into a missing-path error from `uv sync` inside `GraspKit-Tools/`, symlink or rename so a lowercase `graspkit` path exists. The same applies to `rCSFs/` if any tooling references it as `rcsfs`.
- Editing `GraspKit/src/...` is immediately visible to `GraspKit-Tools` (no rebuild needed) once `uv sync` has been run in Tools.
- **`rcsfs` is consumed as a prebuilt wheel, not as an editable source dep.** The source under `rCSFs/` is the upstream of the wheels in `GraspKit-Tools/package/rcsfs/`. Editing Rust or Python in `rCSFs/` does **not** propagate to the Tools venv until you rebuild and drop a fresh wheel into that directory:
  ```bash
  cd rCSFs
  maturin build --release        # produces dist/rcsfs-<ver>-cp314-...whl
  cp dist/rcsfs-<ver>-cp314-*.whl ../GraspKit-Tools/package/rcsfs/
  cd ../GraspKit-Tools && uv sync   # pick up the new wheel
  ```
  For tight iteration on `rcsfs` itself, work inside `rCSFs/` with its pixi/maturin develop loop (`pixi shell && maturin develop`), then ship a wheel only once changes are stable.
- Both projects require Python ≥ 3.14 and pin `torch==2.10.0` / `torchvision==0.25.0` via `cpu` / `gpu` extras. The `gpu` extra routes through the official PyTorch CUDA index; `cpu` resolves via the configured Tsinghua/Aliyun mirrors. On macOS the GPU extra is marker-disabled in Tools.

## Working Across Both Repos

When a change touches both library and pipeline (common case), develop inside `GraspKit/` for API/algorithm changes, inside `rCSFs/` for Rust extension changes, and inside `GraspKit-Tools/` for orchestration/config changes — there is no top-level build that spans them. Each repo has its own `pyproject.toml`, lockfile, lint/type config, and tests, and ships its own `.venv` for isolated unit testing. **Day-to-day pipeline work uses the GraspKit-Tools venv** (see Python Environment above); use the per-repo venv only when iterating on that repo's own tests.

The typical end-to-end flow (calculation → descriptors → ML training → CSF selection → re-validation) is orchestrated from `GraspKit-Tools/ml_CSFs_selection_scripts/`, configured via `config.toml` (validate with `uv run python ml_CSFs_selection_scripts/csfs_ml_choosing_config_load.py validate -f config.toml`), and submitted via `run_script/`. The `train` step imports the ML modules from `graspkit` — so failures there often need fixes in `GraspKit/src/graspkit/ml_module/`. The descriptor-generation step calls into `rcsfs` — failures involving CSF parsing, Parquet I/O, or descriptor normalization typically need fixes in `rCSFs/src/` (Rust) or `rCSFs/rcsfs/` (Python wrapper).

## External Dependencies (Not in This Workspace)

- **GRASP2018** — Fortran package providing `rangular_mpi`, `rmcdhf`, `rci`, `jj2lsj`, `rlevels`, etc. Must be on `PATH`. Source: https://github.com/compas/grasp.
- **CSFs_2_descripors** — provides the `csf_descriptor` binary used to turn CSF files into ML features. Source: https://github.com/YenochQin/CSFs_2_descripors.

Neither is vendored here; both must be installed and exposed via `PATH` (and `GRASP_PATH` / `CSFS_DESCRIPTORS_PATH` env vars) before the pipeline will run end-to-end.
