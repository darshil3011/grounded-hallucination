# Build a two-tower conversion candidate-generation system

> **How to use this file.** Put it at the root of an empty repo, open Claude Code, and say:
> *"Read this and build it milestone by milestone, stopping after each for review."* Build in
> order — each milestone is small, testable, and depends only on the ones before it.

## 1. What you're building

A retrieval (candidate-generation) system that, given a user, returns the top-K items they're
most likely to *convert* on (buy), not just click. It trains a two-tower model on a scarce,
noisy conversion signal — borrowing strength from abundant engagement data — encodes users and
items into embeddings, and serves candidates via approximate-nearest-neighbor (ANN) search.

Worked example it should handle: a user browsing cookware → user embedding → ANN over indexed
item embeddings → top-K high-purchase-intent product candidates → downstream ranking.

## 2. Core design principle

Organize everything around the scarce target signal (conversions). Use an abundant proxy
(engagement) as an auxiliary task, weighted so it can't dilute the target. Denoise the proxy
before trusting it. Train at the granularity where the label is stable. Keep the two towers
separate (no user–item interaction features) so item embeddings can be precomputed and
retrieved fast. Optimize the exact embeddings you serve.

## 3. Architecture

```
training data (conversions + duration-weighted clicks + hard negatives, multi-surface)
        │
   ┌────┴─────────────────────────────┐
   ▼                                   ▼
 user tower                          item tower
  features → parallel(DCN v2 ∥ MLP)   features → parallel(DCN v2 ∥ MLP)
   → user embedding                    → item embedding
   └──────────────┬────────────────────┘
                  ▼
   unified multi-task head + loss (conversion + engagement aux + advertiser/group-level)
                  │  (train)
                  ▼
   SERVING: user emb → ANN over indexed item embeddings → top-K candidates → ranking
```

## 4. Tech stack (recommended, swappable)

- Python 3.11+; PyTorch for the model (or your framework of choice).
- An ANN library (FAISS or similar) for retrieval; start with a brute-force stub.
- **Stub-first:** run on tiny synthetic data with a CPU, no real dataset required.
- **No secrets in code**; any data-warehouse credentials come from env vars.
- `pytest`; `ruff`.

## 5. Project structure

```
conv_cg/
  __init__.py
  data/
    schema.py          # Interaction, Example, Label types
    build_examples.py  # positives (conversions + engagement), negatives, reweighting
    synthetic.py       # tiny synthetic dataset so everything runs offline
  features/
    user.py            # context (visual/graph emb) + history (sequence -> encoder)
    item.py            # id + content + performance features
  model/
    cross_layers.py    # parallel DCN v2 + MLP block (NOT stacked)
    towers.py          # user tower, item tower
    heads.py           # unified multi-task head
    losses.py          # sampled-softmax; conversion + engagement(aux) + group-level
  train.py             # training loop, multi-task weighting
  index/
    ann.py             # build item-embedding index; query top-K (brute-force stub -> FAISS)
  serve.py             # user emb -> ANN -> candidates
  eval.py              # recall@K offline metric
  demo.py              # runnable example end to end
tests/
  test_reweighting.py
  test_parallel_cross.py
  test_recall.py
pyproject.toml
README.md
```

## 6. Data models

```
Interaction: user_id, item_id, surface, type: "conversion"|"click"|"repin"|"impression",
             dwell_seconds: float | None, advertiser_id: str
Example: user_features: dict, item_features: dict, label_type: str, weight: float
Label weighting: click weight = log-based function of dwell_seconds capped at t_max
Negative: non-engaged impressions used as hard negatives (plus in-batch negatives)
```

Rule: conversions are the target; engagement is an auxiliary positive with a dwell-based
weight; non-engaged impressions are hard negatives.

## 7. Build plan (milestones)

| # | Milestone | Done when |
|---|-----------|-----------|
| M0 | Scaffold repo, tooling, empty modules | `pytest` runs (0 tests ok) |
| M1 | `data/schema.py` + `data/synthetic.py` | tiny dataset generates offline |
| M2 | `data/build_examples.py`: positives + dwell reweighting + hard negatives | reweighting + negative sampling unit-tested |
| M3 | `features/user.py` + `features/item.py` (context + history + item features) | features produced for synthetic rows |
| M4 | `model/cross_layers.py`: **parallel** DCN v2 + MLP block | test asserts both branches read raw input (not stacked) |
| M5 | `model/towers.py`: user + item towers producing embeddings | towers output fixed-dim embeddings |
| M6 | `model/heads.py` + `model/losses.py`: unified multi-task loss | conversion + engagement-aux + group-level losses combine with weights |
| M7 | `train.py`: multi-task training loop with tunable weights | loss decreases on synthetic data |
| M8 | `index/ann.py`: build item index; brute-force top-K | returns nearest items for a query embedding |
| M9 | `serve.py`: user emb → ANN → candidates | end-to-end query returns top-K |
| M10 | `eval.py`: recall@K; compare stacked vs parallel cross layers | parallel ≥ stacked recall on synthetic (sanity) |
| M11 | `demo.py`: full worked example offline | prints candidates for the cookware example |
| M12 | (stretch) FAISS index + larger synthetic scale | serving latency measured |

## 8. Component specs

**Data building.** Positives = conversions (primary) + engagement (auxiliary, dwell-weighted).
Negatives = in-batch + non-engaged impressions (hard). One dataset spanning all surfaces, with
a surface feature.

**Features.** User = real-time context embeddings (visual/graph) + history sequence through an
encoder. Item = id + content (multimodal) + performance. Keep towers independent: no
user–item cross features at retrieval.

**Parallel cross block.** Both a DCN v2 cross network and an MLP read the *same raw input* in
parallel; concatenate their outputs. Do **not** feed DCN output into the MLP (see the subtle
bit). The cross network references the original input at each layer.

**Towers + heads.** Two towers → embeddings → a *unified* multi-task head so the served
embeddings are the ones the multi-task loss optimizes.

**Losses.** Sampled-softmax retrieval loss for conversion + engagement (auxiliary, weighted) +
a group-level (advertiser-level) objective for a more stable granularity.

**Index + serve.** Precompute item embeddings, build an ANN index, retrieve top-K by user
embedding. Ranking is downstream and out of scope here.

## 9. The subtle bit (do not collapse this)

Do **not** implement the cross layers as `input → DCN v2 → MLP` (stacked). That creates an
information bottleneck: the MLP only sees what the DCN already transformed, so discarded signal
is unrecoverable. Implement them **in parallel** — both the cross network and the MLP consume
the raw input directly, and their outputs are combined — with the cross network re-referencing
the original input at every layer. Add a test that verifies the MLP branch is wired to the raw
input, not to the DCN output.

## 10. Cross-cutting (granularity / train-serve alignment / realistic negatives)

- **Granularity:** add a group-level (advertiser-level) objective; per-item conversion labels
  are too high-variance to learn from alone.
- **Train-serve alignment:** serve the exact embeddings the multi-task objective optimized (a
  unified head), not one head of a multi-head model.
- **Negatives:** include non-engaged impressions as hard negatives so the model learns the real
  served distribution.

## 11. Config (start simple)

No prompts here — it's a model. Keep a small config: embedding dim, tower layer sizes, DCN
depth, multi-task loss weights (conversion vs engagement vs group), dwell `t_max`, top-K.
Validate config before training.

## 12. Conventions & constraints

- No secrets in code; data credentials from env only. Synthetic data is the default; runs on
  CPU with no external dataset.
- Type hints everywhere; unit-test the data logic and the parallel-cross wiring.
- Each component independently testable; the pipeline runs end to end on synthetic data.

## 13. Definition of done

- `python -m conv_cg.demo` runs offline on synthetic data: builds examples (with dwell
  reweighting + hard negatives), trains the two-tower multi-task model, indexes item
  embeddings, and returns top-K candidates for the worked example.
- A test proves the cross block is parallel (MLP reads raw input), and parallel ≥ stacked
  recall on the synthetic sanity check.
- Dwell-based reweighting and hard-negative sampling are unit-tested.
- `pytest` passes; swapping the brute-force index for FAISS changes nothing about serving.

## 14. Stretch goals

- FAISS/ScaNN index and latency benchmarking at larger synthetic scale.
- Sequence encoder (Transformer) for user history; GraphSAGE-style context embeddings.
- Multi-surface training with surface-specific features and per-surface eval.
- Offline↔online metric correlation harness before "shipping" a model.
- Downstream ranking stage and an end-to-end conversion simulation.
