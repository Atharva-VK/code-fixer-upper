# TRACKER: Graph-Aware Code Fixer Agent

Spec: [SPEC.md](./SPEC.md) · Timebox: Thu 11/19 → Sun 11/22 · Reviewer: Mentor

**Status legend:** ⬜ todo · 🟦 in progress · ✅ done · ⛔ blocked · ➖ dropped

## Current status

| | |
|---|---|
| **Phase** | Day 1: Setup & spike |
| **Next up** | T1.1 |
| **Blockers** | None |
| **Last updated** | 2026-11-19 |

---

## Day 1 (Thu): Setup & spike
**Exit check:** hello-world ADK 2.0 `Workflow` runs in `adk web` on Vertex.

| ID | Task | Spec | Status | Notes |
|---|---|---|---|---|
| T1.1 | `uv init`, `.python-version` = 3.13, project layout skeleton | §5 | ⬜ | |
| T1.2 | `uv add` dependencies; pytest `testpaths = ["tests"]` | §10 | ⬜ | |
| T1.3 | `.env.example`, `.gitignore` (`workspace/`, `eval/results/`, `.env`) | §10 | ⬜ | |
| T1.4 | Vertex auth: `gcloud auth application-default login`; confirm model ID in region | §10 | ⬜ | Model ID: |
| T1.5 | **Spike:** 2.0 `Workflow` + 1 LLM node + 1 dummy MCP tool | §14 | ⬜ | |
| T1.6 | **Spike:** confirm it runs in `adk web` | §14 | ⬜ | If no → use fallback (§14) |
| T1.7 | Start sample repo: `shop/` modules, all tests passing (pre-bug) | §6 | ⬜ | |

## Day 2 (Fri): Sample repo, core, MCP server
**Exit check:** `uv run pytest` passes, including MCP smoke test.

| ID | Task | Spec | Status | Notes |
|---|---|---|---|---|
| T2.1 | Finish sample repo (7 modules, 25–40 tests, all green) | §6 | ⬜ | |
| T2.2 | Plant B1 trap (cart passes % to `apply_discount`; loyalty calls correctly) | §6 | ⬜ | |
| T2.3 | Plant B2 cross-module (tax bug surfacing in orders tests) | §6 | ⬜ | |
| T2.4 | Plant B3, B4 edge cases | §6 | ⬜ | |
| T2.5 | Write `eval/bugs.yaml` (ids, failing tests, trap function) | §6 | ⬜ | |
| T2.6 | `codefix_core.sandbox` + tests | §7 | ⬜ | |
| T2.7 | `codefix_core.files` (list/read/replace) + tests | §7 | ⬜ | |
| T2.8 | `codefix_core.testing` (junitxml → `TestReport`, timeout) + tests | §7 | ⬜ | |
| T2.9 | `codefix_core.graph` (AST → networkx, alias resolution) + tests | §8 | ⬜ | |
| T2.10 | `codefix_mcp.server` (6 tools, stdio, `WORKSPACE_ROOT`) | §7 | ⬜ | |
| T2.11 | MCP smoke test (`list_tools` = 6, call `run_tests`) | §11.1 | ⬜ | |
| T2.12 | Sample repo sanity test (failures match `bugs.yaml`) | §11.1 | ⬜ | |
| T2.13 | `scripts/reset_workspace.py` | §5 | ⬜ | |

## Day 3 (Sat): Agent workflow
**Exit check:** end-to-end fix of B1–B4 in `adk web`.

| ID | Task | Spec | Status | Notes |
|---|---|---|---|---|
| T3.1 | `run_tests_node` (calls core directly) | §9 | ⬜ | |
| T3.2 | `analyzer` agent, read-only `tool_filter` | §9 | ⬜ | |
| T3.3 | `fixer` agent + prompt rules (no tests, `get_callers` first) | §9 | ⬜ | |
| T3.4 | `fix_workflow` dynamic node with `MAX_ITERATIONS` cap | §9 | ⬜ | |
| T3.5 | `diff_node` (difflib vs `sample_repo/`) | §9 | ⬜ | |
| T3.6 | `reporter` agent + report format | §9 | ⬜ | |
| T3.7 | `CODEFIX_GRAPH_TOOLS=off` toggle | §10 | ⬜ | |
| T3.8 | Prompt tuning: B1 fixed in caller, not callee | §6 | ⬜ | |
| T3.9 | Manual run in `adk web` passes | §12 | ⬜ | |

## Day 4 (Sun): Eval, docs, demo
**Exit check:** all acceptance criteria met.

| ID | Task | Spec | Status | Notes |
|---|---|---|---|---|
| T4.1 | `scripts/eval.py` (programmatic runner, metrics, JSON output) | §11.2 | ⬜ | |
| T4.2 | Run 5× eval, record results below | §11.2 | ⬜ | |
| T4.3 | README (prereqs, setup, demo, eval, diagram, limitations) | §12 | ⬜ | |
| T4.4 | Fresh-clone test (≤5 min to running) | §12 | ⬜ | |
| T4.5 | Demo rehearsal | — | ⬜ | |
| T4.6 | *Stretch:* graph-off comparison | §11.3 | ⬜ | |

---

## Acceptance criteria (§12)

- [ ] `uv sync && uv run pytest` passes on a fresh clone
- [ ] `reset_workspace` + `adk web agents` → "fix the failing tests" → all pass + report
- [ ] B1–B4 fixed; B1 fixed in `cart.py`, not `apply_discount`
- [ ] No test file modified in any run
- [ ] Loop never exceeds `CODEFIX_MAX_ITERATIONS`
- [ ] Eval ≥ 4/5 successful runs
- [ ] README complete

## Eval results

| Date | Mode | Runs | Success | Avg iterations | Trap triggered | Avg tokens | Avg duration | Notes |
|---|---|---|---|---|---|---|---|---|
| | graph on | | | | | | | |
| | graph off | | | | | | | |

## Decisions log
Decisions made during the build. D1–D10 are in the spec (§3).

| # | Date | Decision | Why |
|---|---|---|---|
| D11 | | | |

## Open questions

| # | Question | Owner | Status | Answer |
|---|---|---|---|---|
| Q1 | OK to use ADK 2.0 dynamic workflow instead of deprecated LoopAgent? | Mentor | Open | |
| Q2 | Does `adk web` support a dynamic-workflow root? | Me (T1.6) | Open | |
| Q3 | Which Gemini model ID is available in our Vertex region? | Me (T1.4) | Open | |

## Daily log

### Thu 11/19
- 

### Fri 11/20
- 

### Sat 11/21
- 

### Sun 11/22
-