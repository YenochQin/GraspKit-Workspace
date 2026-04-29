# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Layout

This directory is a workspace holding two paired repositories that are developed together. Each subproject has its own `CLAUDE.md` with detailed commands and architecture — read it before working inside that subproject.

- **`GraspKit/`** — the **internal development library** (`grasp-kit`, currently `3.2.dev2`). This is where developers write and modify the core APIs: GRASP2018 output processing, CSF processing, ML-driven CSF selection, and plotting. **Not published as a standalone product** — it is consumed only as an editable dependency of GraspKit-Tools. See `GraspKit/CLAUDE.md` and `GraspKit/AGENTS.md`.
  - Source is split across four `src/` packages built into one wheel: `graspkit/` (main: `data_IO`, `CSFs_processor`, `grasp_data_extractor`, `ml_module`, `utils`), `graspkit_config/` (Pydantic config models), `graspkit_ml/`, `graspkit_plot/`.
- **`GraspKit-Tools/`** — the **end-user / publicly released product**. Bundles pipeline scripts, configuration UIs, and SLURM orchestration on top of `grasp-kit`. This is what external users install and run; treat its CLI/config surface as the public contract. See `GraspKit-Tools/CLAUDE.md` and `GraspKit-Tools/AGENTS.md`.
  - Entry points: `ml_CSFs_selection_scripts/` (training, config loader, run scripts, Streamlit config app under `initialization_tools/config_app/`), `pyscript/` (analysis/plotting), `scripts/` (shell helpers), and `package/rcsfs/` (prebuilt `rcsfs` wheels for Linux x86_64 and macOS arm64, cp314).

## Python Environment

**Always activate from `GraspKit-Tools/.venv`, never from `GraspKit/.venv`.** Tools' uv environment installs `grasp-kit` editable from `../graspkit`, so it provides everything Kit's own venv would plus the pipeline dependencies (`rcsfs`, Streamlit, etc.). Running scripts under Kit's venv will miss `rcsfs` and the Tools-only packages.

```bash
cd GraspKit-Tools
uv sync --extra dev --extra cpu     # or --extra gpu on CUDA hosts
source .venv/bin/activate           # or: uv run <cmd>
```

Edits inside `GraspKit/src/...` are picked up immediately by this environment because of the editable install — no reinstall needed.

## Cross-Repo Coupling

`GraspKit-Tools/pyproject.toml` declares `grasp-kit` as an **editable local dependency** at `../graspkit` (lowercase) via `[tool.uv.sources]`, and pulls `rcsfs` wheels from `package/rcsfs/` via `[tool.uv] find-links`. Implications:

- The two repos must sit side-by-side under this workspace directory, and the Tools repo expects the Kit repo at `../graspkit`. The actual directory is named `GraspKit` (CamelCase) — this resolves on case-insensitive filesystems (default macOS APFS) but **will break on case-sensitive Linux**. If running into a missing-path error from `uv sync` inside `GraspKit-Tools/`, symlink or rename so a lowercase `graspkit` path exists.
- Editing `GraspKit/src/...` is immediately visible to `GraspKit-Tools` (no rebuild needed) once `uv sync` has been run in Tools.
- `rcsfs` is a compiled extension shipped only as wheels; there is no source in this workspace. Bumping its version means dropping a new wheel into `GraspKit-Tools/package/rcsfs/`.
- Both projects require Python ≥ 3.14 and pin `torch==2.10.0` / `torchvision==0.25.0` via `cpu` / `gpu` extras. The `gpu` extra routes through the official PyTorch CUDA index; `cpu` resolves via the configured Tsinghua/Aliyun mirrors. On macOS the GPU extra is marker-disabled in Tools.

## Working Across Both Repos

When a change touches both library and pipeline (common case), develop inside `GraspKit/` for API/algorithm changes and inside `GraspKit-Tools/` for orchestration/config changes — there is no top-level build that spans both. Each repo has its own `pyproject.toml`, `uv.lock`, lint/type config, and tests, and Kit ships its own `.venv` for isolated unit testing — but **day-to-day work, including running tests that exercise the pipeline, should use the GraspKit-Tools venv** (see Python Environment above).

The typical end-to-end flow (calculation → descriptors → ML training → CSF selection → re-validation) is orchestrated from `GraspKit-Tools/ml_CSFs_selection_scripts/`, configured via `config.toml` (validate with `uv run python ml_CSFs_selection_scripts/csfs_ml_choosing_config_load.py validate -f config.toml`), and submitted via `run_script/`. The `train` step imports the ML modules from `graspkit` — so failures there often need fixes in `GraspKit/src/graspkit/ml_module/`.

## External Dependencies (Not in This Workspace)

- **GRASP2018** — Fortran package providing `rangular_mpi`, `rmcdhf`, `rci`, `jj2lsj`, `rlevels`, etc. Must be on `PATH`. Source: https://github.com/compas/grasp.
- **CSFs_2_descripors** — provides the `csf_descriptor` binary used to turn CSF files into ML features. Source: https://github.com/YenochQin/CSFs_2_descripors.

Neither is vendored here; both must be installed and exposed via `PATH` (and `GRASP_PATH` / `CSFS_DESCRIPTORS_PATH` env vars) before the pipeline will run end-to-end.
