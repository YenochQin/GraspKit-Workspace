# graspkit Workspace

Development workspace for the **graspkit** toolchain: high-precision atomic
structure calculations that combine **GRASP2018** with **machine learning** to
select the Configuration State Functions (CSFs) that actually matter, instead
of running a full CI expansion that grows exponentially with the number of
electrons.

The end-to-end flow is:

```text
GRASP calculation -> CSF descriptors -> ML training -> CSF selection -> re-validation
```

This repository is a workspace wrapper, not a monorepo. The four project
directories are Git submodules and keep their own independent histories. The
top-level repository only records shared coordination files and the exact
submodule commits used together.

## Projects

| Directory | Package | Version | Role |
| --- | --- | --- | --- |
| `graspkit-tools/` | `graspkit-tools` | `2.3.dev1` | End-user, publicly released product: pipeline scripts, configuration UI, SLURM orchestration, analysis and plotting. |
| `graspkit/` | `grasp-kit` | `3.4.dev1` | Internal development library: GRASP2018 output processing, CSF processing, ML training and selection, plotting. |
| `rCSFs/` | `rcsfs` (extension `_rcsfs`) | `1.3.1` | Rust/PyO3 extension for high-throughput CSF-to-Parquet conversion and descriptor generation. |
| `nist_data/` | `nist-data` | `0.1.0` | Reads and normalizes NIST ASD exports and supplies reference energy values. |

Versions above come from each project's `pyproject.toml` (`Cargo.toml` for
`rCSFs`). Dependencies point in one direction: `graspkit-tools` consumes
`grasp-kit`, `nist-data`, and `rcsfs` as local path dependencies, so the three
supporting projects never import the tools package.

`graspkit-tools` holds the public contract. Treat its CLI, configuration
surface, and generated run scripts as user-facing interfaces. GRASP2018 itself
is an external Fortran package and is not vendored here.

## Requirements

- Python 3.14 or newer; all four projects require `>= 3.14`.
- [uv](https://docs.astral.sh/uv/) for environment and dependency management.
- A Rust toolchain (Rust 2024 edition) and a platform C/C++ linker to build
  `rcsfs`. On Windows, install Visual Studio Build Tools with the **Desktop
  development with C++** workload before the first `uv sync`.
- GRASP2018 installed separately, available on `PATH`, with `GRASP_PATH`
  pointing at the installation. Executables used by the pipeline include
  `rangular_mpi`, `rmcdhf_mpi`, `rci_mpi`, `jj2lsj`, `rlevels`, `rwfnestimate`,
  `rnucleus`, `mkdisks`, and `rsave`.
- GPU PyTorch wheels only on CUDA hosts that need them.

Verify the external pieces with:

```bash
which rangular_mpi
echo "$GRASP_PATH"
```

## Getting Started

All Python work happens in the single shared environment owned by
`graspkit-tools`. Creating it also installs `grasp-kit` and `nist-data` as
editable path dependencies and builds `rcsfs` through maturin.

```bash
cd graspkit-tools
uv sync
```

On CUDA hosts that need GPU PyTorch wheels, use the GPU extra instead:

```bash
cd graspkit-tools
uv sync --no-group cpu --extra gpu
```

Confirm the sibling packages resolve:

```bash
cd graspkit-tools
uv run python -c "import graspkit, rcsfs; print('Packages OK')"
```

The supporting projects sit next to `graspkit-tools` by design. Tools resolves
them at `../graspkit`, `../nist_data`, and `../rCSFs`, so a `uv sync` that
reports a missing path usually means the layout or submodule checkout is
wrong. On case-sensitive filesystems, confirm the lowercase `graspkit` path
exists.

## Development Workflow

Every project in this workspace shares the environment at
`graspkit-tools/.venv`. Do not create a per-project environment, and do not run
`uv venv` or `uv sync` inside `graspkit`, `rCSFs`, or `nist_data`.

| Where you are working | How to run Python |
| --- | --- |
| `graspkit-tools/` | `uv run <command>` |
| sibling projects | `source ../graspkit-tools/.venv/bin/activate`, or call `../graspkit-tools/.venv/bin/python` directly |

Do not run `uv run` from a sibling project: uv may select or create that
project's own environment. An activated environment is the reliable route
there.

Edits under `graspkit/src/...` and `nist_data/src/...` are picked up
immediately through the editable installs. Rust changes in `rCSFs/` are not:
they must be rebuilt before the Tools environment sees them.

```bash
cd rCSFs
source ../graspkit-tools/.venv/bin/activate
maturin develop           # tight iteration on the extension
# or, once changes are stable:
maturin build --release
cd ../graspkit-tools && uv sync
```

Activating the shared environment first matters for Rust work: PyO3 must link
against the Python 3.14 runtime exposed by that venv. Cargo run without it may
discover a system Python and fail at link time.

On Windows, or on any platform where the Rust/C extension needs an external
compiler environment, initialize that environment before any step that
compiles `rCSFs/`. On this Windows machine the nushell config defines an
`msvc` function, so run `msvc` first, then `uv sync`, `maturin build
--release`, or related commands.

## Quality Checks

Run the checks from `graspkit-tools/` using the shared environment:

```bash
uv run pytest                                            # tools tests
uv run pytest ../graspkit/tests ../nist_data/tests        # sibling tests
uv run ruff check . ../graspkit ../nist_data              # lint
uv run basedpyright ml_CSFs_selection_scripts pyscript    # type-check tools
uv run basedpyright ../graspkit/src                       # type-check the library
```

Rust tests use the activated venv as described above:

```bash
cd rCSFs
source ../graspkit-tools/.venv/bin/activate
cargo test
```

Narrow the paths to the projects you actually touched; the commands above are
the full sweep.

## Running the Pipeline

The pipeline is configured through TOML and driven from
`graspkit-tools/ml_CSFs_selection_scripts/`. Start from the annotated template:

```bash
cd graspkit-tools
cp ml_CSFs_selection_scripts/config_with_comments.toml /path/to/calculation/config.toml
```

Validate the result with the same Pydantic model the pipeline uses:

```bash
uv run python -c "from ml_CSFs_selection_scripts.ml_csf_choosing.ml_module import load_config; load_config('/path/to/calculation/config.toml'); print('Config OK')"
```

The helper script reads and edits individual values; it is not a full
validator:

```bash
uv run python ml_CSFs_selection_scripts/csfs_ml_choosing_config_load.py get target.atom -f /path/to/calculation/config.toml
```

Then either launch the configuration UI,

```bash
uv run streamlit run ml_CSFs_selection_scripts/initialization_tools/config_app/app.py
```

or submit the SLURM orchestration script from the repository:

```bash
sbatch ml_CSFs_selection_scripts/run_script/run_script.sh
```

Step control lets you run or resume individual stages: `initial_csfs`,
`sampling_csfs`, `mkdisks`, `rwfnestimate`, `rci`, `jj2lsj`, `rlevels`, and
`train`. See `graspkit-tools/docs/` and `graspkit-tools/dev_docs/` for details.

## Changing Multiple Projects

Develop each change in the project that owns it:

- API and algorithm changes in `graspkit/`
- Rust extension changes in `rCSFs/`
- NIST ASD parsing and normalized schemas in `nist_data/`
- Orchestration, configuration, and CLI integration in `graspkit-tools/`

There is no top-level build spanning the four projects; each keeps its own
source, build metadata, and tests.

Commits stay layered. Commit the change inside each modified submodule first,
with one commit per submodule, then return here and commit the updated
gitlinks. A top-level commit should contain only coordination files and
gitlink updates, never submodule source. Do not leave advanced gitlinks
uncommitted after reporting a commit as complete, and check `git status` in
every modified repository before you finish.

## Cloning and Submodule Authentication

Clone the full workspace with submodules:

```bash
git clone --recurse-submodules https://github.com/YenochQin/graspkit-Workspace.git GraspKit-Workspace
```

If the repository was cloned without submodules, initialize them later:

```bash
git submodule update --init --recursive
```

If a submodule directory is accidentally deleted, restore it at the exact
commit pinned by the workspace instead of re-cloning it:

```bash
git submodule update --init --recursive nist_data
```

A standalone `git clone` checks out the remote default branch rather than the
commit recorded by the workspace. After restoration, `git status` should no
longer report the submodule as deleted.

### Authenticating with SSH

The clone command above and the URLs in `.gitmodules` use HTTPS, so the
private submodules need a GitHub credential helper or a personal access token;
GitHub account passwords are not accepted for Git operations. If you
authenticate with an SSH key, rewrite GitHub HTTPS URLs once:

```bash
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

After that one-time configuration, `git clone`, `git submodule update --init
--recursive`, and `git submodule update --remote` all follow SSH for the public
workspace and the private submodules alike. `.gitmodules` itself stays on
HTTPS, so collaborators without an SSH key are unaffected.

Run the same command inside the repository without `--global` to scope the
rewrite to this workspace only, and verify it with:

```bash
git config --get-regexp '^url\.'
```

## Updating Submodules

To move one submodule to the latest commit on its configured branch:

```bash
git submodule update --remote graspkit-tools
git add graspkit-tools
git commit -m "workspace: update graspkit-tools submodule"
```

To update all configured submodules:

```bash
git submodule update --remote
git status
```

`git submodule update --remote` follows the `branch` recorded for each
submodule in `.gitmodules`, while `git submodule status` shows the branch
currently checked out. Keep those two in sync when a project moves to a new
release branch. Review and commit the changed gitlinks only after the submodule
repositories are in the desired state.

## Tracked Files

This top-level repository uses an allowlist `.gitignore`: everything is
ignored by default, and only coordination files, submodule gitlinks, and
explicitly listed documents are tracked.

Tracked today:

- `README.md`, `AGENTS.md`, `CLAUDE.md`, `.gitignore`, `.gitmodules`
- the submodule gitlinks `graspkit-tools`, `graspkit`, `rCSFs`, `nist_data`
- `doc/csf_descriptor_v2_ml_design.md` and
  `doc/rcsfs_streaming_generation_plan.md`
- the allowlisted notes under `doc/rmcdhf_test/`

Everything else at the top level is local scratch and must not be committed.
In a working checkout that typically includes `data/`, `docs/research/`,
`temp/`, `grasp_2990_NNNP/`, `rmcdhf_test/`, `.vscode/`, `.agents/`,
`.codex/`, and the various caches. Adding a new coordination document means
adding a matching `!` rule to `.gitignore`.

## Documentation Map

| Document | Covers |
| --- | --- |
| `graspkit-tools/README.md` | Product overview, installation, full project structure |
| `graspkit-tools/docs/` | Getting started, installation, user guide, API reference |
| `graspkit-tools/dev_docs/` | Design notes and implementation plans for in-flight work |
| `doc/csf_descriptor_v2_ml_design.md` | CSF descriptor v2 and ML design |
| `doc/rcsfs_streaming_generation_plan.md` | Streaming descriptor generation |
| `AGENTS.md`, `CLAUDE.md` | Repository guidelines for agents working in this workspace |
| each submodule's own `README.md`, `AGENTS.md`, `CLAUDE.md` | Project-specific guidance |

Read the relevant subproject guide before changing code inside it.

## Repository Visibility

`graspkit`, `graspkit-tools`, and `nist-data` are private GitHub repositories,
while this workspace is public. Users without access to the private
repositories can still clone the public workspace, but those submodules will
fail to download. The public workspace exposes their repository names, URLs,
and pinned commit hashes through `.gitmodules` and the gitlinks, but never
their source contents.
