# Build a simulation-and-evaluation flywheel for an LLM chatbot

> **How to use this file.** Put it at the root of an empty repo, open Claude Code, and say:
> *"Read this and build it milestone by milestone, stopping after each for review."* Build in
> order — each milestone is small, testable, and depends only on the ones before it.

## 1. What you're building

A test harness that validates changes to an LLM chatbot **before** production. It generates
realistic multi-turn conversations (an LLM plays the customer), runs them against the chatbot
under test (backed by mocked backend data), grades each conversation with binary LLM-as-judge
checks, and feeds failures back as new scenarios and new checks — a flywheel.

Worked example it should handle: a frustrated customer with a missing grocery item who wants
a refund; the chatbot hallucinates a non-existent policy; the hallucination check fails; after
a context fix it passes across hundreds of simulations.

## 2. Core design principle

LLMs do the parts that require language and judgment — simulating a believable user and
grading conversations. Everything else is ordinary deterministic engineering — orchestration,
running many conversations, storing scenarios and results, mocking backends, computing pass
rates. Keep the LLM surface small and wrap it in deterministic validation and plumbing.

## 3. Architecture

```
production transcripts ──▶ scenario generation (LLM) ──▶ test scenarios (store)
                                                              │
                                   ┌──────────────────────────▼──────────────────────────┐
                                   │ simulation runtime (multi-turn)                       │
                                   │   LLM simulator  ⇄  chatbot under test  ◀── backend   │
                                   │   (plays user)      (system being tested)     mocks   │
                                   └──────────────────────────┬──────────────────────────┘
                                                              ▼
                                              LLM-as-judge (binary checks)
                                                              ▼
                                              results + failure traces
                                                              ▼
                                              fixes: prompt / retrieval / context
                                                              │
        ◀───────── flywheel: new scenarios + new checks, re-run ─────────┘
```

Cross-cutting: judge checks are calibrated against human labels; the loop runs many
conversations in parallel.

## 4. Tech stack (recommended, swappable)

- Python 3.11+, async (`asyncio`) for running many conversations concurrently.
- `pydantic` v2 for scenario / transcript / result schemas (or stdlib `dataclasses`).
- A thin LLM client behind an interface; **key from an environment variable only**. Start
  with a stub client so everything runs offline.
- `pytest` + `pytest-asyncio`; `ruff` for lint/format.

## 5. Project structure

```
eval_flywheel/
  __init__.py
  models.py            # Scenario, BehavioralProfile, Turn, Conversation, CheckResult, RunReport
  llm/
    base.py            # LLMClient interface
    stub.py            # offline canned client (no keys)
    real.py            # real client, key from env (added later)
  scenarios/
    generate.py        # LLM: transcripts -> behavioral profiles -> Scenario
    store.py           # deterministic load/save of scenarios
  simulator.py         # LLM customer: next user turn given scenario + chat history
  chatbot/
    under_test.py      # adapter calling YOUR chatbot; stub returns canned replies
    backend_mocks.py   # deterministic mocked backend data the chatbot reads
  judge.py             # LLM-as-judge: binary checks over a finished Conversation
  runtime.py           # multi-turn loop: simulator <-> chatbot until end/escalation/cap
  orchestrator.py      # run N scenarios in parallel, aggregate, produce RunReport
  flywheel.py          # add failing case -> new scenario + new check, re-run
  demo.py              # runnable example
tests/
  test_runtime.py
  test_judge.py
  test_orchestrator.py
pyproject.toml
README.md
```

## 6. Data models

```
BehavioralProfile: personality: str, narrative: str, desired_outcome: str
Scenario: id: str, profile: BehavioralProfile, backend_state: dict, source_transcript_id: str | None
Turn: role: "user" | "bot", text: str
Conversation: scenario_id: str, turns: list[Turn], ended_reason: str   # resolved | escalated | turn_cap
Check: id: str, description: str          # phrased as a yes/no policy question
CheckResult: check_id: str, passed: bool, rationale: str
RunReport: scenario_results: dict[str, list[CheckResult]], pass_rate: float
```

Rule: a Check is always a single binary question ("did the bot follow policy X?"). No numeric
quality scores.

## 7. Build plan (milestones)

| # | Milestone | Done when |
|---|-----------|-----------|
| M0 | Scaffold repo, tooling, empty modules | `pytest` runs (0 tests ok) |
| M1 | `models.py` + `llm/base.py` + `llm/stub.py` | models validate; stub imported |
| M2 | `scenarios/generate.py` — transcript -> BehavioralProfile -> Scenario (LLM) | a sample transcript yields a Scenario |
| M3 | `scenarios/store.py` — save/load scenarios | round-trips to disk/JSON |
| M4 | `chatbot/under_test.py` + `backend_mocks.py` (stubs) | returns a canned reply given history + mocked data |
| M5 | `simulator.py` — next user turn from scenario + history (LLM) | produces an in-persona reply that reacts to last bot turn |
| M6 | `runtime.py` — multi-turn loop with end conditions (resolved / escalated / turn cap) | yields a full Conversation |
| M7 | `judge.py` — binary checks over a Conversation (LLM) | returns CheckResults; includes a hallucination check |
| M8 | `orchestrator.py` — run N scenarios via `asyncio.gather`, aggregate pass rate | RunReport over many scenarios |
| M9 | `flywheel.py` — turn a failing case into a new scenario + new check, re-run | failure is captured as a permanent regression |
| M10 | `demo.py` — runs the worked example offline against stubs | prints fail -> fix -> pass |
| M11 | `llm/real.py` (key from env) + swap flag | real client optional, stub default |
| M12 | (stretch) human-calibration harness for the judge | judge agreement vs labels reported |

## 8. Component specs

**Scenario generation — LLM.** In: a transcript. Out: a `Scenario` with a behavioral profile
and the backend state implied by the situation. Ground it in the real transcript, not
invented edge cases.

**Simulator — LLM.** In: scenario + conversation so far. Out: the next user `Turn`. Must stay
in persona, react to the bot's last reply, and follow realistic escalation (multiple chances
before asking for a human; re-engage on progress). Deterministic code enforces the turn cap
and end conditions.

**Chatbot under test — adapter.** In: conversation + mocked backend. Out: the bot's next
reply. This wraps the real system you're validating; the stub returns canned replies so the
harness runs offline.

**Backend mocks — deterministic.** Provide the structured data the bot would read live
(orders, delivery status, refunds). Deliberately include the "too much raw context" failure so
the hallucination check has something to catch.

**Judge — LLM.** In: a finished `Conversation` + the set of `Check`s. Out: a `CheckResult` per
check, each strictly binary with a short rationale. Calibrate against human labels before
trusting it.

**Orchestrator — deterministic.** Run many scenarios concurrently (bounded), collect results,
compute pass rate. No model logic here.

## 9. The subtle bit (do not collapse this)

Do **not** implement the simulator as transcript replay. Replaying fixed user turns breaks the
moment the bot deviates from the recorded conversation and can never surface a new path. The
simulator must **generate each user turn in reaction to the bot's actual latest reply**, while
holding a fixed behavioral profile across the conversation. Keep `runtime.py` (the live
multi-turn loop) strictly separate from `scenarios/` (the fixed profile that seeds it).

## 10. Cross-cutting (judging / context / scale)

- **Binary judging:** every check is yes/no; never a 1–10 score. Store rationales for
  debugging, not for scoring.
- **Context discipline:** give the chatbot-under-test a structured "case state" view of backend
  data rather than raw logs; make "context too noisy" one of the failure modes you can test.
- **Parallelism:** `orchestrator.py` runs scenarios concurrently with a bounded semaphore so a
  full run is minutes, not hours.

## 11. Prompts (start simple)

One-line prompts per LLM task to start; always request strict JSON and validate it.

- scenario: "From this support transcript, output a behavioral profile: personality,
  narrative, desired_outcome. Transcript: {transcript}"
- simulator: "You are this customer: {profile}. Conversation so far: {history}. Write the
  customer's next message."
- judge: "Given this conversation and check '{check}', answer pass=true/false with a one-line
  reason. Conversation: {conversation}"

## 12. Conventions & constraints

- No secrets in code; LLM key from env only. Stub client + stub chatbot are the default.
- Type hints everywhere; validate all LLM output before use.
- Every component independently testable; deterministic parts need no LLM.

## 13. Definition of done

- `python -m eval_flywheel.demo` runs offline and prints the worked example: the hallucination
  check fails, a context fix is applied, and the check passes across many simulated runs.
- The simulator reacts to bot replies (not replay); changing a stubbed bot reply changes the
  simulated user's next turn.
- Checks are binary; the orchestrator reports a pass rate over many scenarios.
- `pytest` passes; swapping the stub LLM for the real client changes nothing about the loop.

## 14. Stretch goals

- Human-calibration harness: measure judge agreement against labeled conversations.
- Adversarial scenario injection (prompt-injection, abusive users) weighted toward failure
  recovery.
- A "one-click" UI to launch a run and diff pass rates between two chatbot versions.
- CI gate: block deploys if pass rate drops on the regression scenario set.
