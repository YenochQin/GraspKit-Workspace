# GraspKit Workspace

This repository coordinates the GraspKit project set. It is a workspace wrapper,
not a monorepo: the three project directories are Git submodules and keep their
own independent histories.

## Layout

- `GraspKit-Tools/` is the user-facing library and final externally released
  project. The workspace tracks branch `2.1dev2`.
- `GraspKit/` is the developer-focused Python library for core package
  development. The workspace tracks branch `3.2dev2`.
- `rCSFs/` is the Rust/PyO3 source project for the compiled `rcsfs` Python
  extension used by the tools pipeline. The workspace tracks branch
  `1.2.2-beta1`.

The top-level repository records the exact submodule commits used together.
Changes inside a submodule must be committed and pushed in that submodule first,
then the top-level workspace commit is updated to point at the new submodule
commit.

## Cloning

Clone the full workspace with submodules:

```bash
git clone --recurse-submodules https://github.com/YenochQin/GraspKit-Workspace.git
```

If the repository was cloned without submodules, initialize them later:

```bash
git submodule update --init --recursive
```

`GraspKit` and `GraspKit-Tools` are private GitHub repositories. Users without
access can clone this public workspace, but those two submodules will fail to
download. The public workspace exposes their repository names, URLs, and pinned
commit hashes, but not their source contents.

## Updating Submodules

To move one submodule to the latest commit on its configured branch:

```bash
git submodule update --remote GraspKit-Tools
git add GraspKit-Tools
git commit -m "workspace: update GraspKit-Tools submodule"
```

To update all configured submodules:

```bash
git submodule update --remote
git status
```

Review and commit the changed gitlinks in the top-level workspace after the
submodule repositories themselves are in the desired state.

## Environment

Start Python work from `GraspKit-Tools/`:

```bash
cd GraspKit-Tools
uv sync
uv run pytest
```

`GraspKit-Tools` uses the local `GraspKit` package as an editable dependency.
Changes in `GraspKit/src/` are picked up through the Tools environment after the
submodules have been initialized.

## Tracked Files

This top-level repository tracks shared coordination files and submodule
gitlinks:

- `README.md`
- `AGENTS.md`
- `CLAUDE.md`
- `.gitignore`
- `.gitmodules`
- `GraspKit-Tools`
- `GraspKit`
- `rCSFs`
