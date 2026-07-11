# Build an agentic-RAG agent with a sandbox as working memory

> **How to use this file.** Put it at the root of an empty repo, open Claude Code, and say:
> *"Read this and build it milestone by milestone, stopping after each for review."* Build in
> order — each milestone is small, testable, and depends only on the ones before it.

## 1. What you're building

An agent that completes data-heavy, multi-step tasks over a large corpus by retrieving only
relevant context and then working out of a **sandbox** (a filesystem + code runtime) instead
of stuffing everything into the LLM context window. Retrieval finds the data; the sandbox
holds it; the agent reads files on demand and runs code to analyze them.

Worked example it should handle: "Review all Q4 opportunities and analyze pipeline health" →
the agent retrieves structured + unstructured records, the set exceeds the context window, it
persists the overflow to the sandbox, runs analysis code over the files, and returns a ranked
list with win probabilities.

## 2. Core design principle

Keep two concerns separate: **retrieval** (find the right data) and **memory** (hold it while
working). Retrieval is an explicit step that returns only relevant, high-quality context.
Memory lives in a sandbox filesystem, not the prompt: data the agent needs right now goes in
the context window; everything else is persisted to files and read on demand. Prefer
persistence over deletion. The LLM plans and reasons; deterministic code does retrieval,
ranking, file I/O, sandboxing, and analysis.

## 3. Architecture

```
user task ──▶ agentic engine (plan · execute · adapt, LLM)
                     │  retrieves via
                     ▼
        retrieval & ranking (graph/index, hybrid search, filters, selectors)  ◀── data sources
                     │  returns only relevant context
                     ▼
        ┌─────────── short-term memory ───────────┐
        │  context window (now)   agent sandbox    │   sandbox = filesystem + CLI +
        │      (LLM)        ──overflow──▶ (files)   │   Python runtime + tool index
        └──────────────────────────────────────────┘
                     │  agent reads files / runs code on demand (loop back to engine)
                     ▼
              analysis / ranked report

Cross-cutting: sandbox is isolated (per-session, scoped FS, no network by default) and
inherits the same permissions/governance as all other tool + model calls.
```

## 4. Tech stack (recommended, swappable)

- Python 3.11+, async for the agent loop.
- A thin LLM client behind an interface; **key from an environment variable only**. Stub it
  first so the whole thing runs offline.
- A sandbox abstraction: start with a **local temp directory + subprocess Python runner** as a
  stub; design the interface so it can later be backed by a real isolated VM/container.
- A simple retrieval layer: start with local files + keyword/embedding stub; design for a real
  vector/graph backend later.
- `pytest` + `pytest-asyncio`; `ruff`.

## 5. Project structure

```
sandbox_agent/
  __init__.py
  models.py            # Task, Doc, RetrievalResult, MemoryRef, AnalysisResult
  llm/
    base.py            # LLMClient interface
    stub.py            # offline canned client (no keys)
    real.py            # real client, key from env (later)
  retrieval/
    graph.py           # entity/relationship index linking records (stub)
    search.py          # hybrid search + filters + selector (snippet) tools
  memory/
    context_window.py  # tracks token budget; holds "needed now" items
    sandbox.py         # Sandbox interface: write_file, read_file, list, run_code
    classifier.py      # classify items: user input / system / tool output -> keep vs persist
  engine.py            # agentic loop: plan -> retrieve -> route to memory -> act -> finish
  analysis_tools.py    # code the agent runs in the sandbox (aggregate, rank, outliers)
  security.py          # isolation + permission/governance checks for sandbox ops
  demo.py              # runnable example
tests/
  test_memory_routing.py
  test_sandbox.py
  test_engine.py
pyproject.toml
README.md
```

## 6. Data models

```
Task: id: str, prompt: str
Doc: id: str, source: str, kind: "structured" | "unstructured", content: str, score: float
RetrievalResult: docs: list[Doc], query: str
MemoryRef: location: "context" | "sandbox", key: str, summary: str   # where an item lives
AnalysisResult: summary: str, ranked: list[dict]                      # final output
```

Rule: retrieval returns ranked, filtered `Doc`s only. The router decides per item whether it
goes in the context window (needed now) or is persisted to the sandbox (everything else).

## 7. Build plan (milestones)

| # | Milestone | Done when |
|---|-----------|-----------|
| M0 | Scaffold repo, tooling, empty modules | `pytest` runs (0 tests ok) |
| M1 | `models.py` + `llm/base.py` + `llm/stub.py` | models validate; stub imported |
| M2 | `retrieval/search.py` + `retrieval/graph.py` (stubs): ranked, filtered docs | query returns relevant docs, filters applied |
| M3 | `memory/context_window.py` — token-budget tracker | reports when capacity is exceeded |
| M4 | `memory/sandbox.py` — Sandbox interface + local stub (temp dir + subprocess) | write/read/list/run_code work on disk |
| M5 | `memory/classifier.py` — keep-in-context vs persist-to-sandbox routing | items route correctly by type + budget |
| M6 | `engine.py` — agent loop: plan → retrieve → route → act | runs end to end against stubs |
| M7 | `analysis_tools.py` — code the agent runs over sandbox files (aggregate, rank) | produces a ranked list from files |
| M8 | overflow path: when context exceeds budget, persist remainder to sandbox + read on demand | large corpus handled without overflow |
| M9 | `security.py` — per-session isolation, scoped FS, permission checks on sandbox ops | cross-session access denied in a test |
| M10 | `demo.py` — runs the worked example offline against stubs | prints ranked Q4 opportunities |
| M11 | `llm/real.py` (key from env) + swap flag | real client optional, stub default |
| M12 | (stretch) real sandbox backend (container/VM) behind the same interface | demo runs unchanged on real backend |

## 8. Component specs

**Agentic engine — LLM loop.** Plans the task, calls retrieval, decides what to keep vs
persist, invokes analysis code, and assembles the answer. Keep the LLM responsible for
*decisions*, not for holding all the data.

**Retrieval & ranking — deterministic + ranking.** Hybrid search + filters (date, type,
ranking signals) + a selector tool that returns specific snippets by similarity. Returns only
relevant, ranked docs. A graph module links related records so retrieval can expand along
relationships.

**Context window — deterministic budget tracker.** Holds items the agent needs immediately;
knows its token budget and signals overflow. It does not silently truncate — overflow is a
routing decision, not a drop.

**Sandbox — interface + backend.** `write_file / read_file / list / run_code`. The stub uses a
temp dir + subprocess; the interface must allow a real isolated backend later. This is the
agent's working memory and compute.

**Classifier/router — deterministic.** Labels each item (user input / system instruction /
tool output) and decides keep-in-context vs persist-to-sandbox. Persistence beats deletion.

**Analysis tools — deterministic code run in the sandbox.** Aggregation, outlier detection,
ranking/scoring over files. The agent writes/sequences these; the math is plain code.

**Security — deterministic.** Per-session isolation, scoped filesystem, network off by
default, permission/governance checks on every sandbox operation.

## 9. The subtle bit (do not collapse this)

Do **not** implement "memory" as "make the context window bigger / summarize harder." That
keeps memory tied to model attention, which drifts across model versions. The agent's durable
working state must live in the **sandbox filesystem**, read deterministically, so behavior
doesn't change when the model updates. Concretely: `memory/sandbox.py` (file-backed, stable)
is the system of record for the working set; `memory/context_window.py` only holds what's
needed for the current step. Persisting to files is the default; keeping something in context
is the exception.

## 10. Cross-cutting (security / reuse / both-matter)

- **Security:** isolate per session, scope the filesystem, disable network by default, and run
  every sandbox op through the same permission/governance checks as your other tools.
- **Reuse:** keep your existing retrieval/ranking/classification; drop token-budget
  micromanagement, truncation, and compression hacks.
- **Both matter:** the sandbox doesn't replace good retrieval or a good model — it removes the
  context-window bottleneck between them.

## 11. Prompts (start simple)

One-line prompts per LLM task; always request strict JSON and validate it.

- plan: "Given task {task}, output an ordered list of steps and what data each needs."
- route: "For these retrieved items {items} and current budget {budget}, return which to keep
  in context and which to persist to the sandbox."
- analyze: "Write Python to compute {goal} over the files at {paths}; return code only."

## 12. Conventions & constraints

- No secrets in code; LLM key from env only. Stub LLM + local sandbox are the default.
- Type hints everywhere; validate all LLM output before use.
- Every component independently testable; deterministic parts need no LLM.

## 13. Definition of done

- `python -m sandbox_agent.demo` runs offline and prints the worked example: a corpus larger
  than the context budget is handled by persisting overflow to the sandbox and analyzing files
  with code, producing a ranked result.
- Forcing a small context budget does not drop data — it routes the remainder to the sandbox.
- A cross-session sandbox access attempt is denied in a test.
- `pytest` passes; swapping the stub LLM for the real client (and later the local sandbox for a
  real backend) changes nothing about the engine.

## 14. Stretch goals

- Real isolated sandbox backend (container/microVM) behind the same interface.
- Real vector + graph retrieval backends.
- Selector tools with greedy materialization (load a full doc, revisit sections, run code).
- Permission-aware retrieval so users only ever see authorized data.
- Token-cost and wall-clock tracing per step to find hot spots.
