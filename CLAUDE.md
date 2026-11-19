# Project Navigation
- `docs/SPEC.md`: design spec. Read the relevant section (§) before starting a task.
- `docs/TRACKER.md`: task tracker. Update status/notes after finishing each task, and log any decision that deviates from the spec.

# Commands
- Install: `uv sync` (use `uv add` for new deps; never pip)
- Unit tests: `uv run pytest`
- Reset agent workspace: `uv run python scripts/reset_workspace.py`
- Run agent UI: `uv run adk web agents`
- Eval: `uv run python scripts/eval.py --runs 5`

# Rules
- ADK 2.0 only: use `Workflow` / `@node` dynamic workflows. Do NOT use the deprecated `LoopAgent` / `SequentialAgent`. Check adk.dev docs when unsure; most online examples are 1.x.
- Logic lives in `src/codefix_core`; `codefix_mcp` stays a thin wrapper. No MCP/ADK imports in core.
- `sample_repo/` is the pristine broken repo. Only change it when planting or adjusting bugs, and keep `eval/bugs.yaml` in sync.
- `workspace/` is disposable and gitignored. Never edit it by hand.