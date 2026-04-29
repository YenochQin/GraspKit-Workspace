# GraspKit Workspace

This repository tracks workspace-level metadata and guidance for the paired
GraspKit projects.

## Layout

- `GraspKit/` is the developer-focused library for core package development.
- `GraspKit-Tools/` is the user-facing library and final externally released
  project.

The two project directories keep their own Git histories and are intentionally
excluded from this workspace repository.

## Environment

Start Python work from `GraspKit-Tools/`:

```bash
cd GraspKit-Tools
uv sync
uv run pytest
```

`GraspKit-Tools` uses the local `GraspKit` package as an editable dependency, so
changes in `GraspKit/src/` are picked up through the Tools environment.

## Tracked Files

This top-level repository tracks only shared coordination files:

- `README.md`
- `AGENTS.md`
- `CLAUDE.md`
- `.gitignore`
