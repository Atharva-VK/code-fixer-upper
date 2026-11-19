# SPEC: Graph-Aware Code Fixer Agent

**Exercise:** First training exercise. Build a custom AI agent with Google ADK, backed by a custom MCP server, tested locally, managed with uv.
**Reviewer:** Mentor
**Timebox:** Thursday afternoon through Sunday (about 3½ days)
**Python:** 3.13

---

## 1. Summary

A code fixer agent that repairs a deliberately broken Python repo until its test suite passes. It runs a **fix, test, repeat** loop as an ADK 2.0 dynamic workflow. All file access and test running go through a **custom MCP server**. The server also builds a small **code knowledge graph** (modules, functions, imports, calls) so the agent can check who depends on a function before changing it. That check is what stops it from making a fix that passes one test and breaks another caller.

The project touches three topics the company is working on:

| Topic | Where it appears |
|---|---|
| **Orchestration** | ADK 2.0 dynamic workflow: diagnose, then a bounded fix-and-verify loop, then a report |
| **Harness** | Sandboxed tools, structured test feedback, a deterministic exit condition, and a repeatable eval script |
| **Knowledge graphs** | AST-built call/import graph exposed as MCP tools (`get_callers`, `get_dependencies`) |

## 2. Goals and non-goals

### Goals
1. Show the ADK concepts in practice: LLM agents, a workflow, MCP tools, session/state, and `adk web`.
2. Build an MCP server that is safe (sandboxed), small (6 tools) and independently tested.
3. Fix all planted bugs in the sample repo, including a "trap" bug that needs the graph to avoid.
4. Prove reliability with repeated runs, not a single lucky demo.
5. Let the mentor clone the repo and run it in under 5 minutes.

### Non-goals
- Semantic or LLM-built graph layers, graph analytics, visualisation.
- Classes and methods in the graph (top-level functions only).
- Languages other than Python.
- Remote or HTTP MCP transport, A2A, deployment.
- Fixing real-world repos.

## 3. Key decisions

| # | Decision | Choice | Why |
|---|---|---|---|
| D1 | Target code | Purpose-built sample repo (inventory/orders domain) | The bugs are known and reproducible, and the repo is easy to reset |
| D2 | Model access | Gemini on **Vertex AI** (ADC auth) | Company standard; no API key in `.env` |
| D3 | Orchestration API | **ADK 2.0 dynamic workflow** (`Workflow` + `@node`) | `LoopAgent`/`SequentialAgent` are deprecated in ADK 2.0. A dynamic workflow allows a plain Python loop with a hard cap |
| D4 | Loop exit | **Deterministic**: code runs pytest and checks the result | The model never decides whether it's done. The tests decide |
| D5 | MCP transport | stdio (ADK launches the server as a subprocess) | Nothing extra to run locally |
| D6 | Edit style | Targeted replace (`old` → `new`, must match exactly once) | Small, reviewable diffs; the model doesn't rewrite whole files |
| D7 | Human approval | None. Autonomous run in a disposable copy, then a final report | Safe because of the sandbox; demos run faster |
| D8 | Graph scope | Modules, top-level functions, imports, calls | Enough for impact checks; method resolution is out of scope |
| D9 | Graph freshness | Rebuilt after every successful edit | The repo is small, so a rebuild costs milliseconds and the graph is never stale |
| D10 | UI | `adk web` | Chat plus event/trace view, good for the demo |

## 4. Architecture

```
                         adk web  /  scripts/eval.py
                                   │
                    ┌──────────────▼──────────────┐
                    │  root_agent: Workflow        │
                    │  fix_workflow (dynamic node) │
                    │                              │
                    │  1. run_tests_node  (code)   │
                    │  2. analyzer        (LLM)    │
                    │  3. loop ≤ MAX_ITERATIONS:   │
                    │       fixer         (LLM)    │
                    │       run_tests_node (code)  │
                    │       all passed? → break    │
                    │  4. diff_node       (code)   │
                    │  5. reporter        (LLM)    │
                    └──────┬───────────────┬───────┘
                           │ MCP (stdio)    │ direct import
                    ┌──────▼──────┐   ┌─────▼────────┐
                    │ codefix_mcp │──▶│ codefix_core │  sandbox · files · pytest runner · graph
                    └─────────────┘   └─────┬────────┘
                                            │
                                     workspace/   (disposable copy of sample_repo/)
```

**Layering rule:** all logic lives in `codefix_core` (plain Python, no MCP or ADK imports). `codefix_mcp` is a thin FastMCP wrapper over it. Deterministic workflow nodes (tests, diff) call `codefix_core` directly. LLM agents only reach the code through MCP tools.

## 5. Project layout

```
codefix/
├── pyproject.toml            # uv project; pytest testpaths = ["tests"]
├── .python-version           # 3.13
├── .env.example
├── README.md
├── SPEC.md
├── src/
│   ├── codefix_core/
│   │   ├── sandbox.py        # path resolution and policy
│   │   ├── files.py          # list/read/replace
│   │   ├── testing.py        # pytest runner → TestReport
│   │   └── graph.py          # AST → networkx graph, queries
│   └── codefix_mcp/
│       └── server.py         # FastMCP, stdio
├── agents/
│   └── codefix_agent/        # discovered by `adk web agents`
│       ├── __init__.py
│       ├── agent.py          # root_agent = Workflow(...)
│       └── prompts.py
├── sample_repo/              # pristine broken repo (never modified by the agent)
│   ├── shop/ …
│   └── tests/ …
├── eval/
│   ├── bugs.yaml             # bug manifest (hidden from the agent)
│   └── results/              # eval output (gitignored)
├── scripts/
│   ├── reset_workspace.py
│   └── eval.py
├── tests/                    # unit tests for core + MCP server
└── workspace/                # gitignored; agent's working copy
```

## 6. Sample repo (`sample_repo/`)

A small inventory/orders package with modules that call each other.

```
shop/
  products.py    # Product catalog, lookup by SKU
  inventory.py   # stock levels, reserve/release
  pricing.py     # unit price, money rounding, apply_discount(price, rate)
  loyalty.py     # loyalty-tier discounts → calls pricing.apply_discount correctly (rate as fraction)
  cart.py        # cart lines, subtotal, promo discount → calls pricing.apply_discount
  tax.py         # tax by region
  orders.py      # place_order: cart → inventory → pricing → tax → total
tests/
  test_products.py, test_inventory.py, test_pricing.py,
  test_loyalty.py, test_cart.py, test_tax.py, test_orders.py
```

About 300–500 lines of source and 25–40 tests. Before the bugs are planted, every test passes. Each bug is then planted as a small, documented change.

### Planted bugs (recorded in `eval/bugs.yaml`)

| ID | Type | Failing test(s) | Root cause | Correct fix |
|---|---|---|---|---|
| B1 | **Trap** | `test_cart.py::test_promo_discount` | `cart.py` passes a percentage (`15`) to `pricing.apply_discount(price, rate)`, which expects a fraction (`0.15`) | Fix the **caller** in `cart.py`. Changing `apply_discount` to divide by 100 makes the cart test pass but breaks `test_loyalty.py`, because `loyalty.py` already calls it correctly. `get_callers("shop.pricing.apply_discount")` shows this. |
| B2 | **Cross-module** | `test_orders.py::test_order_total_*` | The bug is in `tax.py` (e.g. wrong rate lookup or rounding order); the failing test is in orders | Fix `tax.py`. The agent has to trace from orders to tax using `get_dependencies` and `get_callers` |
| B3 | **Edge case** | `test_inventory.py::test_reserve_rejects_non_positive` | `inventory.reserve()` accepts `qty <= 0` | Raise `ValueError` |
| B4 | **Edge case** | `test_cart.py::test_empty_cart_subtotal` | `cart.subtotal()` returns `None` or crashes on an empty cart | Return `0` (or a `Decimal("0.00")` if money is Decimal) |

`bugs.yaml` contains, for each bug, its ID, its failing tests and its **trap function** (B1: `shop.pricing.apply_discount`), so the eval can tell whether the agent fell into the trap.

## 7. MCP server (`codefix_mcp`)

- **Framework:** FastMCP from the official `mcp` Python SDK.
- **Transport:** stdio, launched by ADK through `McpToolset(StdioConnectionParams(...))` with `uv run` (or the venv python) as the command.
- **Scope:** every tool works on `WORKSPACE_ROOT`, passed by env var at launch. There is no tool to change it.

### Tools

| Tool | Args | Returns | Notes |
|---|---|---|---|
| `list_files` | — | `list[str]` relative `.py` paths | Excludes caches and hidden dirs |
| `read_file` | `path: str` | file text with line numbers | Max 2,000 lines |
| `replace_in_file` | `path, old, new` | `{ok, diff}` (unified diff of the change) | `old` must match **exactly once** (error if 0 or >1 matches). **Writes to `tests/` are rejected.** Rebuilds the graph on success. |
| `run_tests` | `selector: str \| None` | `TestReport` (below) | Runs pytest in the workspace; 60 s timeout |
| `get_callers` | `function: str` | `[{caller, file, line}]` | Accepts a qualified name (`shop.pricing.apply_discount`) or a bare name. With a bare name it returns matches for every function of that name |
| `get_dependencies` | `module: str` | `{imports: [...], imported_by: [...]}` | Module-level edges |

### `TestReport` (structured, short)
```json
{
  "passed": 31, "failed": 3, "errors": 0, "all_passed": false,
  "failures": [
    {"test": "tests/test_cart.py::test_promo_discount",
     "message": "assert Decimal('-150.00') == Decimal('85.00')",
     "location": "tests/test_cart.py:42",
     "traceback": "<last ~15 lines>"}
  ],
  "truncated": false
}
```
Implementation: run `python -m pytest` with `--junitxml` to a temp file, using `cwd=WORKSPACE_ROOT` and `-p no:cacheprovider`, then parse the XML. Never return raw pytest output. List at most 10 failures and set `truncated` if there are more.

### Sandbox policy (`codefix_core.sandbox`)
- Resolve every path against `WORKSPACE_ROOT` and reject anything that resolves outside it, including `..`, absolute paths and symlinks that escape.
- Writes are allowed only for `*.py` files outside `tests/`.
- Tools return errors as structured messages (`{ok: false, error: "..."}`), never as stack traces, so the model can recover.

## 8. Code graph (`codefix_core.graph`)

- **Build:** walk `*.py` in the workspace (excluding `tests/`) and parse each file with `ast` into a `networkx.DiGraph`.
- **Nodes:** `module` (e.g. `shop.cart`) and `function` (qualified name, e.g. `shop.pricing.apply_discount`, with file and line).
- **Edges:** `module -IMPORTS-> module`, `module -DEFINES-> function`, `function -CALLS-> function`.
- **Call resolution (best effort):**
  - `name()` defined in the same module
  - `name()` brought in by `from shop.x import name` (including `as` aliases)
  - `mod.name()` where `mod` comes from `import shop.x as mod` / `from shop import x`
  - Anything else is left unresolved and counted (the stats are exposed for debugging).
- **Lifecycle:** built when the server starts and rebuilt after every successful `replace_in_file`.
- **Acceptance:** on the pristine sample repo, `get_callers("shop.pricing.apply_discount")` returns both `shop.cart.*` and `shop.loyalty.*` callers.

## 9. Agent workflow (`agents/codefix_agent`)

The root is an ADK 2.0 `Workflow` whose entry node is a dynamic `fix_workflow` node (`@node(rerun_on_resume=True)`) that calls child nodes with `ctx.run_node(...)`.

```python
# Shape only; check the exact API against the ADK 2.0 docs.
@node(rerun_on_resume=True)
async def fix_workflow(ctx, node_input):
    report = await ctx.run_node(run_tests_node, None)
    if report.all_passed:
        return "All tests already pass."
    diagnosis = await ctx.run_node(analyzer, render(report))
    for i in range(MAX_ITERATIONS):                 # default 5
        await ctx.run_node(fixer, render(diagnosis, report))
        report = await ctx.run_node(run_tests_node, None)
        if report.all_passed:
            break
    diff = await ctx.run_node(diff_node, None)       # workspace vs sample_repo, via difflib
    return await ctx.run_node(reporter, render(report, diff, i + 1))

root_agent = Workflow(name="codefix", edges=[("START", fix_workflow)])
```

### Nodes

| Node | Type | Tools (MCP `tool_filter`) | Responsibility |
|---|---|---|---|
| `run_tests_node` | code | none (calls `codefix_core.testing`) | Ground truth for the loop exit |
| `analyzer` | LLM | `list_files`, `read_file`, `get_callers`, `get_dependencies` (**read-only**) | For each failure, find the root-cause function and list its callers. Output: a structured diagnosis |
| `fixer` | LLM | read-only tools + `replace_in_file` | Apply the smallest fix for the diagnosis |
| `diff_node` | code | none | Build a unified diff of the workspace against the pristine repo |
| `reporter` | LLM | none | Write the final report |

### Fixer rules (in the prompt)
1. Never edit tests. The tool enforces this; the prompt explains why.
2. **Before changing a function's behaviour or signature, call `get_callers` on it.** If other callers depend on the current behaviour, fix the caller instead.
3. Make one focused change per bug and do not refactor.
4. If the same fix has failed twice, re-read the code instead of repeating it.

### Final report (reporter output)
- Outcome: all tests pass / N failures remain
- Iterations used out of `MAX_ITERATIONS`
- Per change: file, function, what changed, why, and which callers were checked
- The diff, generated by code and not by the LLM

## 10. Configuration

`.env.example`:
```
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1        # or global
CODEFIX_MODEL=gemini-2.5-flash           # confirm an available model ID in your Vertex region
CODEFIX_MAX_ITERATIONS=5
CODEFIX_GRAPH_TOOLS=on                   # off = hide get_callers/get_dependencies (eval comparison)
```
Auth: `gcloud auth application-default login`. Do **not** set `GOOGLE_API_KEY`.

Dependencies (managed with `uv add`): `google-adk` (≥2.0, with the MCP extra if one is required), `mcp`, `networkx`, `pytest`, `python-dotenv`, `pyyaml`.

## 11. Testing and evaluation

### 11.1 Unit tests (`tests/`, no LLM): **required**
- **Sandbox:** rejects `..`, absolute paths, escaping symlinks, writes to `tests/`, and non-`.py` writes.
- **`replace_in_file`:** errors on 0 matches and on more than 1 match; the success case returns a diff.
- **Test runner:** on a tiny fixture repo with known pass/fail results, the report counts and failure names are correct, and the timeout works.
- **Graph:** known callers and imports on the pristine sample repo, including both callers of `apply_discount`; aliases resolve.
- **MCP smoke test:** start the server over stdio with an MCP client, check that `list_tools` returns exactly the 6 tools, and call `run_tests`.
- **Sample repo sanity:** the pristine repo has exactly the failures listed in `bugs.yaml`.

Run with `uv run pytest`.

### 11.2 Eval script (`scripts/eval.py`): **required**
`uv run python scripts/eval.py --runs 5 [--no-graph]`

For each run: reset the workspace, run the workflow programmatically (an ADK runner without the UI), collect events, then record:

| Metric | Source |
|---|---|
| `success` | final `run_tests` reports `all_passed` |
| `iterations` | loop count |
| `trap_triggered` | any `replace_in_file` that touched the trap function from `bugs.yaml` |
| `tests_untouched` | test files hash the same as the pristine repo (should always be true) |
| `tool_calls` | count per tool |
| `tokens` | sum of event usage metadata |
| `duration_s` | wall clock |

Output: a summary table in the terminal plus `eval/results/<timestamp>.json`.

### 11.3 Graph-off comparison: **stretch goal**
Run the eval with `CODEFIX_GRAPH_TOOLS=off` and with it on, and compare `trap_triggered` rate, iterations and tokens. Report the real numbers even if the difference is small; the model may sometimes avoid the trap just by reading files.

## 12. Acceptance criteria (definition of done)

- [ ] `uv sync && uv run pytest` passes on a fresh clone.
- [ ] `uv run python scripts/reset_workspace.py && uv run adk web agents` runs, and asking "fix the failing tests" ends with all tests passing and a final report.
- [ ] B1–B4 are fixed, and B1 is fixed **in the caller** (`cart.py`), not in `apply_discount`.
- [ ] No test file is modified in any run.
- [ ] The loop never exceeds `CODEFIX_MAX_ITERATIONS`.
- [ ] The eval script reports ≥4/5 successful runs.
- [ ] The README covers prerequisites (uv, gcloud, Vertex access), setup, demo steps, eval usage, the architecture diagram and known limitations.

## 13. Plan

| When | Work | Exit check |
|---|---|---|
| **Thu (rest of day)** | `uv init`, dependencies, `.env`, Vertex auth. **Spike:** a hello-world ADK 2.0 `Workflow` with one LLM node and one MCP tool, running in `adk web`. Start the sample repo. | Hello-world works in `adk web` on Vertex |
| **Fri** | Finish the sample repo and plant the bugs; write `bugs.yaml`. Build `codefix_core` (sandbox, files, testing, graph), the MCP server and the unit tests. | `uv run pytest` passes; MCP smoke test passes |
| **Sat** | Workflow: test node, analyzer, fixer loop, diff, reporter. Tune prompts against B1–B4. | End-to-end fix in `adk web` |
| **Sun** | Eval script, 5-run eval, README, demo rehearsal. Stretch: graph-off comparison. | Acceptance criteria met |

## 14. Risks and open questions

| Risk | Mitigation |
|---|---|
| ADK 2.0 is recent; tutorials and examples mostly show 1.x | Follow adk.dev 2.0 docs only; the Thursday spike validates the API early |
| `adk web` support for a dynamic-workflow root is unconfirmed | Thursday spike. Fallback: run the workflow via the eval script for the loop, and use `adk web` with a single fixer `LlmAgent` for the interactive demo |
| `McpToolset` inside workflow agent nodes behaves differently | Covered by the spike |
| Model ID not available in your Vertex region | Make it configurable (`CODEFIX_MODEL`); check Model Garden on Thursday |
| Agent edits tests to make them pass | Blocked in the tool, checked in the eval (`tests_untouched`) |
| Graph misses calls (dynamic dispatch, aliases) | Keep the sample repo to simple call styles; expose unresolved-call stats; document the limitation |
| Nondeterministic results | Eval over 5 runs; report the rate, not a single run |

**Open question for mentor:** the workflow-agents link in the exercise points to template workflows, which are deprecated in ADK 2.0. This spec uses the 2.0 dynamic workflow instead. Confirm that's acceptable.