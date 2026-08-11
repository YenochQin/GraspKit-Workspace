# graspkit Workspace

This repository coordinates the graspkit project set. It is a workspace wrapper,
not a monorepo: the four project directories are Git submodules and keep their
own independent histories.

## Layout

- `graspkit-tools/` is the user-facing library and final externally released
  project (`2.3.dev1`).
- `graspkit/` is the developer-focused Python library for core package
  development (`3.4.dev1`).
- `rCSFs/` is the Rust/PyO3 source project for the compiled `rcsfs` Python
  extension used by the tools pipeline (`1.3.1`).
- `nist_data/` is the independent `nist-data` Python package for reading and
  normalizing NIST ASD exports. `graspkit-tools` consumes it as an editable
  path dependency, while `grasp-kit` remains independent of NIST-specific I/O.

The top-level repository records the exact submodule commits used together.
Changes inside a submodule must be committed and pushed in that submodule first,
then the top-level workspace commit is updated to point at the new submodule
commit.

## Cloning

Clone the full workspace with submodules:

```bash
git clone --recurse-submodules https://github.com/YenochQin/graspkit-Workspace.git GraspKit-Workspace
```

If the repository was cloned without submodules, initialize them later:

```bash
git submodule update --init --recursive
```

If one submodule directory is accidentally deleted, restore it at the exact
commit pinned by the workspace. For example:

```bash
git submodule update --init --recursive nist_data
```

Do not replace this command with a standalone `git clone`: a normal clone checks
out the remote repository's default branch instead of the commit recorded by
the workspace. After restoration, `git status` should no longer report the
submodule as deleted.

### Authenticating with SSH

The clone command above and the submodule URLs recorded in `.gitmodules` use
HTTPS. Accessing the private `graspkit`, `graspkit-tools`, and `nist-data`
submodules therefore requires a GitHub credential helper or a personal access
token; GitHub account passwords are not accepted for Git operations. Users who
authenticate with an SSH key can configure Git once to rewrite every GitHub
HTTPS URL to SSH:

```bash
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

After this one-time configuration, `git clone`, `git submodule update
--init --recursive`, and `git submodule update --remote` all follow SSH for
the public workspace and the private submodules alike. `.gitmodules` itself
stays on HTTPS, so collaborators without an SSH key are unaffected.

To scope the rewrite to this workspace only, run the same command inside the
repository without `--global`. To verify it is active:

```bash
git config --get-regexp '^url\.'
```

`graspkit`, `graspkit-tools`, and `nist-data` are private GitHub repositories.
Users without access can clone this public workspace, but those submodules will
fail to download. The public workspace exposes their repository names, URLs,
and pinned commit hashes, but not their source contents.

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

Review and commit the changed gitlinks in the top-level workspace after the
submodule repositories themselves are in the desired state.

## Environment

Start Python work from `graspkit-tools/`:

```bash
cd graspkit-tools
uv sync
uv run pytest
```

`graspkit-tools` uses the local `graspkit` package as an editable dependency.
It also uses `nist-data` editable from `../nist_data`. Changes in either package
are picked up through the Tools environment after the submodules have been
initialized.

When working on Windows, or on any platform where building Rust/C extensions
requires an external linker toolchain, initialize that compiler/linker
environment before running `uv sync` or build commands that touch `rCSFs/`. For
example, on this Windows machine the nushell config defines an `msvc` function;
run `msvc` first so the shell can find MSVC tools such as `link.exe`, then run
`uv sync`, `uv run maturin build --release`, or related compile steps from
`rCSFs/`.

## Tracked Files

This top-level repository tracks shared coordination files and submodule
gitlinks:

- `README.md`
- `AGENTS.md`
- `CLAUDE.md`
- `.gitignore`
- `.gitmodules`
- `graspkit-tools`
- `graspkit`
- `rCSFs`
- `nist_data`
