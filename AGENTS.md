# Repository Guidelines

## Project Structure & Module Organization
This workspace contains two related Python projects. `GraspKit-Tools/` is the final externally released, user-facing library. `GraspKit/` is the developer-focused library for ongoing package development. `GraspKit/` source code is under `GraspKit/src/`, with modules such as `graspkit/data_IO/`, `graspkit/CSFs_processor/`, `graspkit/ml_module/`, and shared utilities in `graspkit/utils/`. `GraspKit/tests/` holds pytest tests, and `GraspKit/docs/` holds design notes and change logs.

`GraspKit-Tools/` contains workflow scripts and examples that depend on `grasp-kit`. Main code lives in `GraspKit-Tools/ml_CSFs_selection_scripts/`, analysis helpers in `GraspKit-Tools/pyscript/`, shell helpers in `GraspKit-Tools/scripts/`, and tests in `GraspKit-Tools/tests/`.

## Build, Test, and Development Commands
Start the Python environment from `GraspKit-Tools/`; its `uv` environment links to the local editable `GraspKit/` package. Run most commands from `GraspKit-Tools/` unless building or checking `GraspKit/` itself.

- `uv sync --extra dev --extra cpu`: install development dependencies with CPU PyTorch.
- `uv sync --extra dev --extra gpu`: install the CUDA/GPU variant where supported.
- `uv run pytest`: run the project test suite.
- `uv run ruff check .`: run lint checks.
- `uv run mypy ../GraspKit/src` from `GraspKit-Tools/`: run strict type checks for the developer package.
- `uv run mypy ml_CSFs_selection_scripts pyscript` in `GraspKit-Tools/`: type-check tool code.
- `uv run python -m build` or `python build_package.py --clean` in `GraspKit/`: build package artifacts.

## Coding Style & Naming Conventions
Use Python 3.14+, 4-space indentation, and explicit type hints for public functions. Prefer `pathlib.Path` for filesystem work. Use `snake_case` for modules, functions, variables, and TOML keys; `PascalCase` for classes and Pydantic models; `UPPER_SNAKE_CASE` for constants. Ruff uses `NPY201`; formatting uses double quotes.

## Testing Guidelines
Tests use `pytest`. Add new tests under each project’s `tests/` directory with filenames `test_*.py` and functions named `test_*`. Include realistic sample inputs for loaders, parsers, config normalization, and ML pipeline behavior. Run `uv run pytest` before submitting changes.

## Commit & Pull Request Guidelines
Git history includes short subjects such as `update ...`, `bug fix`, and scoped messages like `fix: clear inherited venv state...`. Prefer concise, scoped, imperative subjects, for example `data_IO: fix binary loader bounds`. PRs should include purpose, affected paths, test results, linked issues or notes, and screenshots for UI changes.

## Security & Configuration Tips
Do not commit credentials, cluster paths, local datasets, generated calculation outputs, or virtual environments. Treat TOML config and shell-script generation inputs as untrusted; validate paths and avoid unsafe shell interpolation.
