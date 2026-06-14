# Repository Guidelines

## Project Structure & Module Organization
This workspace contains three paired repositories that are developed together as Git submodules, not ordinary nested directories. Each subproject has its own `CLAUDE.md` and `AGENTS.md`; read the relevant subproject guide before working inside it.

- `graspkit/` is the internal development library (`grasp-kit`, currently `3.2.dev2`). It contains the core APIs for GRASP2018 output processing, CSF processing, ML-driven CSF selection, and plotting. It is not published as a standalone product; `graspkit-tools` consumes it as an editable dependency. Source is split across four `src/` packages built into one wheel: `graspkit/` (`data_IO`, `CSFs_processor`, `grasp_data_extractor`, `ml_module`, `utils`), `graspkit_config/` (Pydantic config models), `graspkit_ml/`, and `graspkit_plot/`.
- `rCSFs/` is the internal Rust/Python extension for high-throughput CSF-to-Parquet conversion and descriptor generation. The compiled module name is `_rcsfs`, the public Python package is `rcsfs`, and the project uses Maturin/PyO3, Rust 2024, and Python 3.14 only. It is consumed by `graspkit-tools` as a path dependency through `[tool.uv.sources]`.
- `graspkit-tools/` is the end-user, publicly released product. It bundles pipeline scripts, configuration UIs, and SLURM orchestration on top of `grasp-kit` and `rcsfs`. Treat its CLI and config surface as the public contract. Main entry points are `ml_CSFs_selection_scripts/` (training, config loader, run scripts, and the Streamlit config app under `ml_CSFs_selection_scripts/initialization_tools/config_app/`), `pyscript/` (analysis and plotting), and `scripts/` (shell helpers).

The top-level `graspkit-Workspace` repository records coordination files and exact submodule gitlinks. Do not copy submodule source into the parent repository or remove nested `.git` metadata. Make source changes inside the relevant submodule, commit and push them there, then return to the parent workspace and commit the updated gitlink.

## Build, Test, and Development Commands
Start the day-to-day Python environment from `graspkit-tools/`. Its `uv` environment installs `grasp-kit` editable from `../graspkit` and includes the pipeline dependencies such as `rcsfs` and Streamlit. Do not debug `graspkit/` with a separate `graspkit/.venv`; scripts run under that environment will miss Tools-only dependencies.

```bash
cd graspkit-tools
uv sync
source .venv/bin/activate
```

Use `uv run <cmd>` instead of activation when convenient. On CUDA hosts that need GPU PyTorch wheels, run `uv sync --no-group cpu --extra gpu`.

When debugging `graspkit/` or `graspkit-tools/`, use the `uv` environment inside `graspkit-tools/`. Edits under `graspkit/src/...` are picked up immediately because of the editable install. When debugging `rCSFs/`, use the `uv` environment inside `rCSFs/`, because that repository owns the Rust/PyO3 extension build and local extension test loop.

On Windows, or on any platform where Rust/C extension builds need an external compiler/linker environment, initialize that environment before `uv sync` or any compile step that touches `rCSFs/`. For example, this Windows machine uses nushell with an `msvc` function in the shell config; run `msvc` first so the shell can find MSVC tools such as `link.exe`, then run `uv sync`, `maturin build --release`, or related build commands.

- `git clone --recurse-submodules https://github.com/YenochQin/graspkit-Workspace.git GraspKit-Workspace`: clone the workspace and all accessible submodules.
- `git submodule update --init --recursive`: initialize submodules after a normal clone.
- `git submodule status`: show pinned submodule commits.
- `git submodule update --remote <path>`: advance a submodule to the latest commit on its configured branch, then commit the changed gitlink in the parent repository.
- `uv run pytest`: run the active project test suite.
- `uv run ruff check .`: run lint checks.
- `uv run mypy ../graspkit/src` from `graspkit-tools/`: type-check the developer package.
- `uv run mypy ml_CSFs_selection_scripts pyscript` in `graspkit-tools/`: type-check tool code.
- `uv run python -m build` or `python build_package.py --clean` in `graspkit/`: build package artifacts.
- `maturin build --release` in `rCSFs/`: build platform wheels before copying or resolving them into the Tools environment.

## Cross-Repo Coupling
`graspkit-tools/pyproject.toml` declares `grasp-kit` as an editable local dependency at `../graspkit` and `rcsfs` as a path dependency at `../rCSFs` via `[tool.uv.sources]`.

- The repositories must sit side by side under this workspace directory. Tools expects the Kit repo at `../graspkit`. If running on a case-sensitive filesystem and `uv sync` reports a missing path, ensure the lowercase `graspkit` path exists. Apply the same care if any tooling references `rCSFs/` as `rcsfs/`.
- Editing `graspkit/src/...` is immediately visible to `graspkit-tools` after `uv sync`; no rebuild is needed.
- Editing Rust code in `rCSFs/` requires rebuilding before changes are visible in the Tools venv:
  ```bash
  cd rCSFs
  maturin build --release
  cd ../graspkit-tools && uv sync
  ```
- For tight iteration on `rcsfs` itself, work inside `rCSFs/` with `maturin develop`, then build a release wheel once changes are stable.
- Both Python projects require Python >= 3.14 and pin `torch==2.10.0` / `torchvision==0.25.0` through CPU/GPU extras. The GPU extra uses the official PyTorch CUDA index; the CPU environment resolves through the configured Tsinghua/Aliyun mirrors. On macOS the GPU extra is marker-disabled in Tools.

When a change touches both library and pipeline, develop API/algorithm changes in `graspkit/`, Rust extension changes in `rCSFs/`, and orchestration/config changes in `graspkit-tools/`. There is no top-level build spanning all three projects. Each repository has its own `pyproject.toml`, lockfile, lint/type config, tests, and local `.venv`; day-to-day pipeline work uses the `graspkit-tools` venv, while per-repo venvs are for that repository's own tests.

The normal end-to-end flow is calculation -> descriptors -> ML training -> CSF selection -> re-validation. It is orchestrated from `graspkit-tools/ml_CSFs_selection_scripts/`, configured via `config.toml`, validated with `uv run python ml_CSFs_selection_scripts/csfs_ml_choosing_config_load.py validate -f config.toml`, and submitted through `run_script/`. The `train` step imports ML modules from `graspkit`; failures there often require fixes in `graspkit/src/graspkit/ml_module/`. Descriptor generation calls into `rcsfs`; failures involving CSF parsing, Parquet I/O, or descriptor normalization usually require changes in `rCSFs/src/` or `rCSFs/rcsfs/`.

## Coding Style & Naming Conventions
Use Python 3.14+, 4-space indentation, and explicit type hints for public functions. Prefer `pathlib.Path`. Use `snake_case` for modules, functions, variables, and TOML keys; `PascalCase` for classes and Pydantic models; `UPPER_SNAKE_CASE` for constants. Ruff uses `NPY201`; formatting uses double quotes.

## Testing Guidelines
Tests use `pytest`. Add new tests under each project's `tests/` directory with filenames `test_*.py` and functions named `test_*`. Include realistic sample inputs for loaders, parsers, config normalization, and ML pipeline behavior. Run `uv run pytest` before submitting changes, plus the repository-specific Rust or type-check commands when the affected project needs them.

## Commit & Pull Request Guidelines
Git history includes short subjects such as `update ...`, `bug fix`, and scoped messages like `fix: clear inherited venv state...`. Prefer concise, scoped, imperative subjects, for example `data_IO: fix binary loader bounds`. For workspace-only changes, use subjects such as `workspace: update graspkit submodule`. PRs should include purpose, affected paths, test results, linked issues, and screenshots for UI changes.

Keep parent and submodule commits separate. A parent workspace commit should only update coordination files such as `README.md`, `AGENTS.md`, `.gitmodules`, or submodule gitlink entries. A submodule code commit belongs in that submodule repository.

## Security & Configuration Tips
Do not commit credentials, cluster paths, local datasets, generated calculation outputs, virtual environments, or copied private submodule source. Treat TOML config and shell-script generation inputs as untrusted; validate paths and avoid unsafe shell interpolation.

`graspkit` and `graspkit-tools` are private GitHub repositories. The public workspace may expose their repository names, URLs, and pinned commit hashes through `.gitmodules` and gitlinks, but it must not contain their source contents outside the submodules. Users without access can clone the public parent repository, but private submodule downloads will fail.

GRASP2018 is an external Fortran package providing `rangular_mpi`, `rmcdhf`, `rci`, `jj2lsj`, `rlevels`, and related tools. It must be installed, available on `PATH`, and exposed through `GRASP_PATH` before the pipeline can run end to end.
