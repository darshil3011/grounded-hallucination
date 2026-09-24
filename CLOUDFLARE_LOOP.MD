# Build an agentic multi-agent code review system

> **How to use this file.** Put it at the root of an empty repo, open Claude Code, and say:
> *"Read this and build it milestone by milestone, stopping after each for review."* Build in
> order — each milestone is small, testable, and depends only on the ones before it.

## 1. What you're building

A CI-native system that reviews a merge/pull request automatically. A coordinator agent plans
the review and spawns up to seven specialist sub-agents (security, performance, code quality,
docs, release, compliance, agent-instructions), each narrowly scoped. The coordinator then
judges their findings and posts a single verdict (approve / comment / block). It routes work by
risk tier, survives provider outages, and stays quiet on purpose (few, high-signal findings).

Worked example it should handle: an MR that changes an auth module and 60 files → classified
`full` → 7 agents → a critical authorization finding → coordinator blocks the merge with one
structured comment; on re-push, an incremental re-review resolves it and approves.

## 2. Core design principle

One big prompt produces noise. Split the review into scoped specialists, each told what to flag
AND what to ignore, with a strong coordinator judging their output. LLMs review and judge;
deterministic code does plugins, scheduling, risk tiering, resilience, and routing. Bias toward
approval; keep a human override.

## 3. Architecture

```
MR opened ─▶ CI entry (plugins assemble config)
                     ▼
        diff filter + risk tier (trivial / lite / full)
                     ▼
        coordinator agent (top-tier model, plans)  ◀── control plane
                     ▼                                   (model routing in a
        spawn_reviewers (scheduler: concurrency,          config store; kill-switch;
        timeouts, circuit breakers, failback)             telemetry)
                     ▼
   ┌───── up to 7 specialists, parallel (own prompt + model) ─────┐
   │ security · performance · code quality · docs · release ·     │
   │ compliance · agent-instructions → structured findings (XML)  │
   └──────────────────────────────┬───────────────────────────────┘
                     ▼
        coordinator judge pass (dedupe · re-categorize · filter · severity)
                     ▼
        decision → VCS action (approve / comment / unapprove / block; break-glass)
```

## 4. Tech stack (recommended, swappable)

- TypeScript/Node or Python; async for concurrent sessions.
- A coding-agent runtime with an SDK that supports programmatic sessions (the reference system
  uses OpenCode). **Stub it first:** a fake agent runtime that returns canned findings so the
  whole pipeline runs offline with no LLM and no real repo.
- A VCS adapter behind an interface (start with a local git diff + a fake "post comment").
- **No secrets in code**; model/API keys and VCS tokens come from env vars.
- `pytest`/`vitest`; linter.

## 5. Project structure

```
ai_review/
  models.ts|py         # MR, DiffEntry, Finding, ReviewResult, RiskTier
  plugins/
    base                # ReviewPlugin interface: bootstrap / configure / postConfigure
    vcs_gitlab          # VCS provider (stub: local git + fake comment poster)
    ai_gateway          # model tiers + failback chains
    compliance          # internal-standards checks
    telemetry           # fire-and-forget tracking
  diff/
    filter              # strip lock/minified/generated files
    risk                # classify trivial / lite / full
  agent/
    runtime             # coding-agent runtime interface + stub
    coordinator         # plan + judge passes
    specialists/        # one scoped prompt per domain (what to flag + what NOT to)
    spawn_reviewers     # scheduler: concurrency, timeouts, circuit breakers, failback
  resilience/
    circuit_breaker     # closed/open/half-open per model tier
    errors              # classify errors -> shouldFailback
  control_plane/
    routing             # fetch model routing from a config store; provider kill-switch
  security/
    prompt_guard        # strip prompt-boundary tags from user content
  stream/
    jsonl               # parse JSONL events incrementally
  demo                  # runnable example
tests/
pyproject/package.json
README.md
```

## 6. Data models

```
DiffEntry: path, addedLines, removedLines, isGenerated, isSecuritySensitive
RiskTier: "trivial" | "lite" | "full"
Finding: reviewer, severity: "critical"|"warning"|"suggestion", path, line, message
ReviewResult: findings: Finding[], decision: "approved"|"approved_with_comments"|
              "minor_issues"|"significant_concerns", action
```

Rule: findings are structured and severity-tagged (not prose); the decision is derived from
them by a fixed rubric.

## 7. Build plan (milestones)

| # | Milestone | Done when |
|---|-----------|-----------|
| M0 | Scaffold repo, tooling, empty modules | tests run (0 ok) |
| M1 | `models` + `agent/runtime` interface + stub runtime | stub returns canned findings |
| M2 | `plugins/base` + a VCS stub (local git diff, fake comment) | a diff is loaded; a comment is "posted" to a buffer |
| M3 | `diff/filter` + `diff/risk` | noise stripped; MR classified trivial/lite/full (unit-tested) |
| M4 | `agent/specialists`: scoped prompts with explicit "what NOT to flag" | each returns structured, severity-tagged findings |
| M5 | `agent/spawn_reviewers`: run N specialists concurrently by tier | tier selects agent count; runs in parallel |
| M6 | `stream/jsonl`: parse incremental events | partial/truncated stream still yields valid events |
| M7 | `agent/coordinator`: plan + judge (dedupe, filter, severity, rubric) | produces one decision from many findings |
| M8 | `resilience/circuit_breaker` + `errors` + failback chains | open circuit routes to fallback; only retryable errors fail back |
| M9 | `control_plane/routing`: model routing from a config store + kill-switch | disabling a provider reroutes without code change |
| M10 | `security/prompt_guard`: strip boundary tags; break-glass override | injected tags removed; "break glass" forces approval |
| M11 | incremental re-review (aware of prior findings) | fixed findings auto-resolve; unfixed re-emit |
| M12 | `demo`: full worked example offline | prints block → re-review → approve |

## 8. Component specs

**Plugins.** A `ReviewPlugin` interface with bootstrap (concurrent, non-fatal), configure
(sequential, fatal), postConfigure. Plugins contribute config through a context API; no plugin
reads another's internals. VCS coupling isolated in one file.

**Diff filter + risk tier.** Strip lock files, minified/bundled assets, and generated files
(but keep DB migrations). Classify by lines/files changed and security-sensitive paths; tier
sets agent count and model tier.

**Specialists.** One scoped prompt per domain. Each prompt has a "What to flag" and an explicit
"What NOT to flag" section. Output is severity-tagged structured findings. Agents may read/grep
the repo independently.

**spawn_reviewers (scheduler).** Launch specialists concurrently, each its own session/model.
Per-task timeout, overall cap, inactivity detection, retry budget.

**Coordinator.** Plan pass decides which specialists to run. Judge pass dedupes, re-categorizes,
drops speculative/contradicted findings (verify against source when unsure), assigns severity,
and applies the approval rubric. Bias toward approval.

**Resilience.** Circuit breaker per model tier; failback chain to older generation; error
classifier so only retryable API errors fail back.

**Control plane.** Model routing fetched from a config store (KV-like); flipping a provider off
reroutes running jobs fast. Telemetry is fire-and-forget and never blocks.

**Security.** Strip prompt-boundary tags from user-authored MR content before building prompts;
support a `break glass` human override, detected before the review starts.

## 9. The subtle bit (do not collapse this)

Do **not** prompt each agent with "find all the issues." That produces a firehose of vague,
often-wrong findings that users learn to ignore. Every specialist prompt MUST include an
explicit **"What NOT to flag"** section (no theoretical risks, no defense-in-depth nags, nothing
in unchanged code, no "consider library X"). Optimize for few, high-signal findings — target
roughly ~1 finding per review, not ten. Add a test asserting each specialist prompt contains a
"what not to flag" section, and that a clean diff yields zero findings.

## 10. Cross-cutting (resilience / control plane / cost / security)

- **Resilience:** circuit breakers + failback + error classification.
- **Control plane:** routing/kill-switch in a config store; fire-and-forget telemetry.
- **Cost:** don't duplicate full context per agent — write a shared context/patch file agents
  read; rely on prompt caching.
- **Security:** strip boundary tags; human break-glass override.

## 11. Prompts (start simple, but scoped)

Each specialist prompt = "What to flag" + "What NOT to flag" + "output structured findings with
severity." The coordinator prompt = "given these findings, dedupe/filter/judge and decide
approve/comment/block per this rubric." Always parse/validate structured output.

## 12. Conventions & constraints

- No secrets in code; keys/tokens from env only. Stub runtime + stub VCS are the default.
- Type hints/types everywhere; validate all agent output before use.
- Every component independently testable; the pipeline runs end to end offline.

## 13. Definition of done

- `demo` runs offline against stubs: filters a diff, classifies risk, runs the tier's agents,
  judges, and posts one decision; the worked example blocks then approves on re-review.
- A clean diff produces zero findings; each specialist prompt has a "what NOT to flag" section
  (tested).
- Disabling a provider in the config store reroutes without code changes; only retryable errors
  fail back (tested).
- Tests pass; swapping the stub runtime for a real coding-agent SDK changes nothing about the
  orchestration.

## 14. Stretch goals

- Real coding-agent runtime (e.g. OpenCode) + real VCS (GitLab/GitHub) adapters.
- Local mode: run the same agents/prompts on a working-tree diff from the CLI.
- Prompt-cache-aware shared context files; token/cost dashboards.
- An agent-instructions (AGENTS.md) reviewer that flags stale AI instructions.
- Re-review conversation handling ("won't fix" / "I disagree").
