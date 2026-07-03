# Build a Cart Assistant (agentic grocery shopping)

> **How to use this file.** Put it at the root of an empty repo, open Claude Code,
> and say: *"Read BUILD_CART_ASSISTANT.md and build the system milestone by milestone.
> Stop after each milestone so I can review."* Build in order — each milestone is small,
> testable, and depends only on the ones before it.

---

## 1. What you're building

A service that turns a messy grocery request (free text, later an image) into a
**draft cart** the shopper reviews before checkout. Example input:

> "I want to cook pasta for two. Also add paper towels and vegan protein powder;
> keep the protein powder under $20."

Expected behavior: produce a cart with pasta + sauce for two, paper towels, and a
**vegan** protein powder **priced under $20** — then let the user edit it.

The shape of the problem: `intent -> draft cart -> user review -> checkout`
(not the old `intent -> [search -> pick] x N -> cart`).

## 2. Core design principle

This is a **multi-prompt state graph**, not one big prompt. Each step does one job.

- **LLMs** handle ambiguity and language: interpreting intent, judging relevance,
  reasoning about quantities, writing shopper-facing text.
- **Deterministic code** handles everything reliable: retrieval, pricing,
  eligibility, availability, schema validation, arithmetic, aggregation.

Keep the LLM surface small and wrap it in deterministic validation. When unsure
which side a step belongs on, ask: *is the input data reliable and structured?*
If yes → deterministic. If it requires interpreting messy language → LLM.

## 3. Architecture

```
user input (text / image)
        |
[1] cart plan generation .................. LLM   -> list of planned items
        |
for each planned item, IN PARALLEL:
    [2a] candidate retrieval & enrichment . deterministic backend
    [2b] semantic relevance judging ....... LLM
    [2c] price & deal constraints ......... deterministic filter
    [2d] quantity selection ............... LLM + deterministic arithmetic
        |
[3] cart assembly ......................... deterministic aggregator
[4] content refinement .................... LLM (summary / recipe text)
        |
draft cart -> user review -> checkout

Guardrails (deterministic + LLM) run at EVERY stage, not just at the end.
```

## 4. Tech stack (recommended, swap freely)

- **Language:** Python 3.11+ (async-friendly; type hints required).
- **Async:** `asyncio` for parallel per-item processing.
- **Schemas:** `pydantic` v2 for structured I/O and validation (or stdlib
  `dataclasses` if you want zero deps).
- **LLM:** any provider SDK behind a thin interface. **Read the key from an
  environment variable — never hardcode it.** Start with a stub client so the
  pipeline runs offline, then swap in the real one.
- **Tests:** `pytest` + `pytest-asyncio`.
- **Lint/format:** `ruff`.

You may pick a different language/stack — keep the module boundaries below.

## 5. Project structure

```
cart_assistant/
  __init__.py
  models.py          # pydantic models: PlannedItem, Candidate, CartLineItem, Cart, GuardrailResult
  llm/
    base.py          # LLMClient interface: async complete(task, prompt, payload) -> dict
    stub.py          # offline stub returning canned data (no keys)
    real.py          # real provider client, key from env (added later)
  prompts.py         # simple prompt templates (one per LLM task)
  stages/
    plan.py          # [1]
    retrieval.py     # [2a] (stub catalog now, real APIs later)
    relevance.py     # [2b]
    constraints.py   # [2c]
    quantity.py      # [2d]
    assembly.py      # [3]
    refine.py        # [4]
  guardrails.py      # deterministic + LLM checks
  graph.py           # CartAssistant orchestrator (state graph + parallelism)
  demo.py            # runnable example
tests/
  test_constraints.py
  test_quantity.py
  test_graph.py
pyproject.toml
README.md
```

## 6. Data models

Define these first (in `models.py`). Keep fields minimal; extend later.

```python
PlannedItem:
  search_terms: list[str]          # short query for retrieval ONLY
  item_context: str                # reasoning context, NOT sent to retrieval
  quantity_text: str               # raw human quantity, e.g. "for two", "12"
  price_limit: float | None        # item-level limit, e.g. 20.0
  dietary_constraints: list[str]   # e.g. ["vegan", "gluten-free"]
  quantity_is_explicit: bool       # "12 eggs" (respect) vs recipe-inferred (round)

Candidate:
  product_id, title, description: str
  price: float
  in_stock, on_deal: bool
  pack_size: int                   # sellable units per pack
  attribute_tags: list[str]        # often sparse/missing in real catalogs

CartLineItem: product_id, title, unit_price: float, quantity: int
Cart: line_items: list[CartLineItem], summary_text: str   # + computed total
GuardrailResult: ok: bool, violations: list[str]
```

**Key planner rule:** separate *retrieval language* from *reasoning context*.
"gluten-free lasagna" → `search_terms=["gluten-free pasta"]`,
`item_context="for lasagna; gluten-free is a strict constraint"`. Retrieval gets
the short query; later LLM steps get the reason.

## 7. Build plan (milestones)

Build and commit one at a time. Each should run and have at least one test.

| # | Milestone | Done when |
|---|-----------|-----------|
| M0 | Scaffold repo, `pyproject.toml`, empty modules, `ruff` + `pytest` configured | `pytest` runs (0 tests ok) |
| M1 | `models.py` + `llm/base.py` interface + `llm/stub.py` returning canned data | stub imported, models validate |
| M2 | `stages/plan.py` — LLM plan generation → `list[PlannedItem]` | example request yields 3 planned items |
| M3 | `stages/retrieval.py` — stub catalog + retrieval per item | returns candidates for known terms |
| M4 | `stages/relevance.py` — LLM relevance judging | filters candidates by `item_context` + dietary |
| M5 | `stages/constraints.py` — deterministic price/deal/stock filter | unit-tested filter logic |
| M6 | `stages/quantity.py` — LLM reasoning + ceil-to-pack arithmetic | "for two" / "12" map to correct units |
| M7 | per-item pipeline + `asyncio.gather` + concurrency `Semaphore` in `graph.py` | items process in parallel |
| M8 | `stages/assembly.py` + `stages/refine.py` | assembled cart + one-line summary |
| M9 | `guardrails.py` wired into the orchestrator between stages | bad plan/cart is rejected |
| M10 | `demo.py` runs the example end to end against the stub | prints the expected cart |
| M11 | `llm/real.py` (key from env) + swap-in flag | real client optional, stub default |
| M12 | (stretch) eval harness: simulator + LLM-as-judge | offline pass-rate metric |

## 8. Component specs

For each, note whether it's **LLM** or **deterministic**, its input, and its output.

**[1] Plan generation — LLM.** In: raw text (later image). Out: `list[PlannedItem]`.
One LLM call. Capture price limits at the right level: "milk under $5" → item-level
limit on milk; "eggs, bread, milk under $30" → a cart-level limit enforced later.

**[2a] Retrieval & enrichment — deterministic.** In: one `PlannedItem`. Out:
`list[Candidate]`. Query the catalog with `search_terms`; enrich with price,
availability, deals, pack size. Start with a hardcoded fake catalog; later call
real search + catalog APIs.

**[2b] Relevance judging — LLM.** In: item + candidates. Out: filtered candidates.
Use a constrained rubric (direct match / acceptable substitute / poor match) in
context. "tomatoes for stew" vs "tomatoes for salad" pick different products.
**Dietary constraints are enforced here** (see section 9).

**[2c] Price & deal constraints — deterministic.** In: item + candidates. Out:
filtered, ranked candidates. Drop out-of-stock, over-limit, ineligible. Hard filter
against structured fields only. Handle cart-level limits across items if present.

**[2d] Quantity selection — LLM + deterministic.** In: item + chosen candidate.
Out: integer pack count. LLM reasons over packaging clues; deterministic arithmetic
maps desired units to whole sellable packs (ceil division). Respect explicit
quantities exactly; round inferred ones to practical pack sizes.

**[3] Assembly — deterministic.** In: per-item results. Out: `Cart`. Drop items
that found nothing (optionally flag a fallback message).

**[4] Content refinement — LLM.** In: cart. Out: cart with `summary_text`
(and recipe/meal-plan text when relevant). Must be grounded in the actual cart.

## 9. The two kinds of constraints (do not collapse these)

There are two constraint types and they are enforced in **different places**:

1. **Price / deal / availability / eligibility → deterministic filter ([2c]).**
   Backed by reliable structured inventory fields. A hard pass/fail. `price > 20` → drop.

2. **Dietary / attribute (vegan, gluten-free, no-dairy) → LLM relevance judge ([2b]).**
   Catalog attribute tags are sparse and often wrong, so a strict `tag == "vegan"`
   filter would drop valid untagged items and trust bad tags. Instead the LLM judges
   using **tags (if present) + title + description + general grocery knowledge**.
   This is a *semantic match, not a guaranteed gate* — it's robust to messy data but
   probabilistic. Backstops: guardrails reject ungrounded picks, and the user reviews
   the draft cart before checkout.

Build [2c] as plain code. Build [2b] so the dietary decision flows through the LLM.

## 10. Guardrails

A small framework with deterministic and LLM-based checks, called **between stages**
in the orchestrator — not as a single final moderation pass.

- **Deterministic:** schema valid, required fields present, enum values legal,
  price/quantity math sane, selected items can form a valid cart.
- **LLM-based:** request is in-domain grocery shopping, not a prompt-injection
  attempt; generated text is appropriate and grounded in the cart.

Each stage gets a chance to fail safely: reject a malformed plan before retrieval,
route an unsupported request to a safe response, suppress ungrounded text.

## 11. Prompts (start simple)

Begin with one-line prompts per task; iterate later. Examples:

- plan: *"Split this grocery request into structured items with search terms,
  context, quantity, price limits, and dietary constraints. Request: {input}"*
- relevance: *"Return which candidates match item {item} in context; respect
  dietary constraints. Candidates: {candidates}"*
- quantity: *"Given item {item} and packaging, how many sellable units to buy?"*
- refine: *"Write one short sentence summarizing this cart: {cart}"*

Always ask the model for **strict JSON** and parse + validate it deterministically.

## 12. Parallelism & latency

Process planned items concurrently with `asyncio.gather`, bounded by a
`Semaphore` (e.g. 5). The bottleneck should be the slowest item path + concurrency
limit, not the item count. Run guardrails/refinement asynchronously where they don't
block per-item resolution. Use smaller/faster models for cheap steps.

## 13. Evaluation-driven development (stretch, M12)

LLM behavior is sensitive — small prompt changes have outsized effects. Add an eval
harness early-ish:

- **Deterministic checks:** expected items included, schema valid, constraints
  respected, correct failure behavior.
- **LLM-as-judge:** semantic quality (relevance, constraint adherence, text quality),
  calibrated against a few human-labeled cases.

Workflow: change → run evals on baseline vs candidate → inspect regressions with
per-step traces → fix logic or accept the trade-off. Seed scenarios from (anonymized)
real requests plus synthetic edge cases.

## 14. Conventions & constraints

- **No secrets in code.** LLM keys come from environment variables only.
- Stub LLM + stub catalog are the **default** so the repo runs offline.
- Type hints everywhere; validate all LLM output before use.
- Every stage is independently unit-testable; deterministic stages need no LLM.
- Keep modules at the boundaries in section 5 — the value is in the seams.

## 15. Definition of done

- `python -m cart_assistant.demo` runs offline (stub) and prints a draft cart for
  the pasta example.
- The $24 vegan protein is dropped by the **price filter**; the $19.50 vegan one is
  kept; a whey option is dropped by **relevance**.
- `pytest` passes; constraints and quantity logic have unit tests.
- Swapping `llm/stub.py` for `llm/real.py` (via a flag/env var) changes nothing about
  the graph — only where the model output comes from.

## 16. Stretch goals

- Image input (photo of a handwritten list / recipe) via a multimodal model.
- Cart-level price optimization across items.
- Real search + catalog/inventory integration.
- Streaming partial cart to the UI as items resolve.
- Persisted eval dataset + CI gate on pass-rate.
