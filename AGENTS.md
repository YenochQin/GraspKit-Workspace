# Repository Guidelines

## Project Structure & Module Organization
This workspace contains three related projects as Git submodules, not ordinary nested directories. `GraspKit-Tools/` is the final externally released, user-facing library and currently tracks branch `2.1dev2`. `GraspKit/` is the developer-focused Python library and currently tracks branch `3.2dev2`. `rCSFs/` is the Rust/PyO3 source project for the compiled `rcsfs` Python extension used by the tools pipeline and currently tracks branch `1.2.2-beta1`. `GraspKit/` source code is under `GraspKit/src/`, with modules such as `graspkit/data_IO/`, `graspkit/CSFs_processor/`, `graspkit/ml_module/`, and `graspkit/utils/`.

`GraspKit-Tools/` contains workflow scripts and examples that depend on `grasp-kit` and `rcsfs`. Main code lives in `GraspKit-Tools/ml_CSFs_selection_scripts/`, helpers in `pyscript/` and `scripts/`, user docs in `docs/`, and internal notes in `dev_docs/`. Tests live in each project’s `tests/` directory when present.

The top-level `GraspKit-Workspace` repository records coordination files and the exact submodule gitlinks. Do not copy submodule source into the parent repository or remove nested `.git` metadata. Make source changes inside the relevant submodule, commit and push them there, then return to the parent workspace and commit the updated gitlink.

## Build, Test, and Development Commands
Start the Python environment from `GraspKit-Tools/`; its `uv` environment links to local workspace dependencies and installs the wheel-backed `rcsfs` extension. Run most commands from `GraspKit-Tools/` unless building or checking `GraspKit/` or `rCSFs/` directly.

- `git clone --recurse-submodules https://github.com/YenochQin/GraspKit-Workspace.git`: clone the workspace and all accessible submodules.
- `git submodule update --init --recursive`: initialize submodules after a normal clone.
- `git submodule status`: show the pinned submodule commits.
- `git submodule update --remote <path>`: advance a submodule to the latest commit on its configured branch, then commit the changed gitlink in the parent repository.
- `uv sync`: install the default CPU/runtime/dev/UI/docs environment.
- `uv sync --no-group cpu --extra gpu`: install the CUDA PyTorch variant.
- `uv run pytest`: run the project test suite.
- `uv run ruff check .`: run lint checks.
- `uv run mypy ../GraspKit/src` from `GraspKit-Tools/`: type-check the developer package.
- `uv run mypy ml_CSFs_selection_scripts pyscript` in `GraspKit-Tools/`: type-check tool code.
- `uv run python -m build` or `python build_package.py --clean` in `GraspKit/`: build package artifacts.
- `maturin build --release` in `rCSFs/`: build platform wheels before copying them into `GraspKit-Tools/package/rcsfs/`.

## Coding Style & Naming Conventions
Use Python 3.14+, 4-space indentation, and explicit type hints for public functions. Prefer `pathlib.Path`. Use `snake_case` for modules, functions, variables, and TOML keys; `PascalCase` for classes and Pydantic models; `UPPER_SNAKE_CASE` for constants. Ruff uses `NPY201`; formatting uses double quotes.

## Testing Guidelines
Tests use `pytest`. Add new tests under each project’s `tests/` directory with filenames `test_*.py` and functions named `test_*`. Include realistic sample inputs for loaders, parsers, config normalization, and ML pipeline behavior. Run `uv run pytest` before submitting changes.

## Commit & Pull Request Guidelines
Git history includes short subjects such as `update ...`, `bug fix`, and scoped messages like `fix: clear inherited venv state...`. Prefer concise, scoped, imperative subjects, for example `data_IO: fix binary loader bounds`. For workspace-only changes, use subjects such as `workspace: update GraspKit submodule`. PRs should include purpose, affected paths, test results, linked issues, and screenshots for UI changes.

Keep parent and submodule commits separate. A parent workspace commit should only update coordination files such as `README.md`, `AGENTS.md`, `.gitmodules`, or the submodule gitlink entries. A submodule code commit belongs in that submodule repository.

## Security & Configuration Tips
Do not commit credentials, cluster paths, local datasets, generated calculation outputs, or virtual environments. Treat TOML config and shell-script generation inputs as untrusted; validate paths and avoid unsafe shell interpolation.

`GraspKit` and `GraspKit-Tools` are private GitHub repositories. The public workspace may expose their repository names, URLs, and pinned commit hashes through `.gitmodules` and gitlinks, but it must not contain their source contents outside the submodules. Users without access can clone the public parent repository, but private submodule downloads will fail.
