# Build a company-wide knowledge AI agent platform

> **How to use this file.** Put it at the root of an empty repo, open Claude Code, and say:
> *"Read this and build it milestone by milestone, stopping after each for review."* Build in
> order — each milestone is small, testable, and depends only on the ones before it.

## 1. What you're building

An always-on AI agent for non-engineers that already knows your company. Users start a chat
session and it produces artifacts (reports, dashboards, docs) that evolve in place. It reaches
your internal tools and data through a large, federated library of skills, works across long
multi-day sessions, and is reachable from multiple surfaces (web, chat, browser extension)
through one API.

Worked example it should handle: a sales rep asks for a renewal deal summary → the agent
applies their role's config and permissions → selects the right skills → loads only the tools
those skills need → queries data, runs analysis in a sandbox, writes files → returns a
doc + dashboard that the rep edits in place.

## 2. Core design principle

Buy the hard agent infrastructure (tool-calling loop, state, middleware) off the shelf; spend
your effort on the company-specific parts — security, data, and skills. Make the agent a
service behind one API; surfaces are thin views. Don't load all tools into context — let the
model pick skills, and let skills gate which tools load. Give the agent a persistent filesystem
for state, and run code in a sandbox the agent calls as a tool (agent stays outside it).

## 3. Architecture

```
surfaces: web app · chat · browser extension
                     │  (all call one API)
                     ▼
            Agent API (agent as a service)
                     ▼
  configuration layer (per-team agents: skill sets · personas)
                     ▼
  company harness (security · permissions · internal services)
                     ▼
  agent base (tool-calling loop · streaming · state)
     middleware: [filesystem (persistent)] [sandbox (code exec as a tool)] [summarization]
                     ▼
  skills (federated, many teams) ── gate ──▶ tools (hundreds; loaded on demand)
                     ▼
  company systems: data warehouse · chat · docs
                     ▼
  session artifacts: reports · dashboards · docs (evolve across turns)

Cross-cutting: governance — permissions, pinned policy skills, audit, enterprise scale.
```

## 4. Tech stack (recommended, swappable)

- An off-the-shelf agent harness that provides the tool-calling loop, middleware, streaming,
  and state (the reference system uses LangChain's Deep Agents). **Stub it first** so the
  platform runs offline with a fake LLM and fake tools.
- Python is fine; async for long sessions.
- A persistent object store for the virtual filesystem (local dir as a stub → S3 later).
- A sandbox runner for code execution (local subprocess stub → real isolated sandbox later).
- **No secrets in code**; data/warehouse credentials from env only.
- `pytest`; linter.

## 5. Project structure

```
kai/
  models.py              # Session, Message, Skill, Tool, Artifact, UserProfile
  api/
    service.py           # the single agent API (surfaces are clients of this)
  surfaces/
    web.py / chat.py / extension.py   # thin adapters -> api/service.py
  config/
    agents.py            # per-team agent configs (skill sets, personas)
  harness/
    security.py          # permissions, policy, identity
    services.py          # connectors to internal systems (stubbed)
  agent/
    base.py              # agent harness interface + stub (tool-calling loop, state)
    middleware/
      filesystem.py      # persistent virtual FS, sync-in/sync-out
      sandbox.py         # code exec exposed as a TOOL (agent runs outside)
      summarization.py   # long-session context control
  skills/
    registry.py          # federated skill registry (owner, allowedTools, frontmatter)
    selection.py         # LLM picks skills; skills gate tool loading; pinned base skills
  tools/
    registry.py          # hundreds of tools; dynamic loading by allowedTools
  data/
    systems.py           # data warehouse / chat / docs connectors (stubbed)
  demo.py                # runnable example
tests/
pyproject.toml
README.md
```

## 6. Data models

```
UserProfile: user_id, function (e.g. "sales_ops" | "finance"), extra_skills: list
Skill: id, owner_team, description, allowedTools: list[str], pinned: bool, frontmatter(<=1KB)
Tool: id, description, permission_scope
Session: id, user_id, agent_config_id, messages: list[Message], files: VFSHandle
Artifact: id, kind ("doc"|"dashboard"|"report"), content_ref   # lives beside the session
```

Rule: tools are never all loaded at once. Skills carry `allowedTools`; selecting a skill is
what brings its tools into context. A small set of skills is `pinned` (always on).

## 7. Build plan (milestones)

| # | Milestone | Done when |
|---|-----------|-----------|
| M0 | Scaffold repo, tooling, empty modules | tests run (0 ok) |
| M1 | `models.py` + `agent/base.py` interface + stub harness | stub runs a tool-calling loop with canned output |
| M2 | `api/service.py` — one agent API; a web + chat surface adapter call it | both surfaces hit the same API and get the same result |
| M3 | `config/agents.py` — per-team agent configs (skills, persona) | switching config changes available skills/persona |
| M4 | `harness/security.py` — permissions + identity gating | a tool call is denied when out of scope (tested) |
| M5 | `agent/middleware/filesystem.py` — persistent VFS + sync-in/out | files persist across turns in a session |
| M6 | `agent/middleware/sandbox.py` — code exec exposed as a tool | agent calls sandbox tool; agent itself runs outside it |
| M7 | `skills/registry.py` + `tools/registry.py` — federated skills with allowedTools | skills and tools register; ownership tracked |
| M8 | `skills/selection.py` — LLM picks skills → gate tool loading; pinned base | only selected skills' tools load; pinned always present |
| M9 | `agent/middleware/summarization.py` — long-session context control | a long session stays under a token budget |
| M10 | artifacts: session produces/evolves reports/docs/dashboards | artifact created and updated across turns |
| M11 | `demo.py` — full worked example offline against stubs | prints the sales-rep deal-summary flow |
| M12 | (stretch) real harness + real sandbox + warehouse connectors | demo runs unchanged on real backends |

## 8. Component specs

**Agent API (service).** The one primitive. All surfaces are clients. Exposes: start session,
send message, stream events, fetch artifacts.

**Configuration layer.** Per-team agent definitions (skill sets, persona, behavior) layered
over the base, without modifying the harness.

**Company harness.** Identity, permissions, policy, and connectors to internal systems. Every
tool call passes through permission checks.

**Agent base (harness).** Tool-calling loop, streaming, state, middleware composition. Keep
behind an interface so a real harness swaps in.

**Filesystem middleware.** A persistent virtual filesystem; materialize files in before a
sandbox call and sync changes out after, so the agent sees one coherent file environment all
session.

**Sandbox middleware.** Code execution (analytics + file processing) exposed to the agent as a
*tool*. The agent runs outside the sandbox and calls into it — do not run the agent inside the
sandbox.

**Skills + selection.** Federated registry; each skill has a description and an `allowedTools`
list. Selection is two-pass: the LLM picks skills from context, then those skills' tools load.
Pin foundational skills. Keep skill frontmatter tiny (~1KB) and cap how many load at once.

**Tools.** Hundreds registered; only loaded when a selected skill allows them.

## 9. The subtle bit (do not collapse this)

Do **not** load all tools into the system prompt. With hundreds of tools that bloats context
and degrades selection. Implement a **two-pass, skill-gated** loader: (1) the LLM selects
relevant skills using the user/task context; (2) each selected skill's `allowedTools` brings in
only its tools. Always include a small set of **pinned** foundational skills so core context
and policy persist. Add a test that selecting a skill loads exactly its `allowedTools` (plus
pinned) and nothing else, and that unselected tools never enter context. Note the ceiling:
quality drops once too many skills load at once, so plan a retrieval/classifier prefilter
before the LLM's final selection at scale.

## 10. Cross-cutting (security / long sessions / governance)

- **Security:** permission checks on every tool call; sandbox as a called tool; pinned policy
  skills.
- **Long sessions:** summarization threshold/model/output-size as explicit cost + quality
  knobs; avoid large-context cache misses.
- **Governance:** audit trails, enterprise permissions, and a path to per-user behavioral
  personalization.

## 11. Prompts / config (start simple)

Keep prompts minimal: a system prompt that establishes company context + pinned skills; a
skill-selection prompt ("given the task and profile, pick the skills to load"). Each skill is a
small structured module (description + allowedTools + how-to). Validate selections before
loading.

## 12. Conventions & constraints

- No secrets in code; credentials from env only. Stub harness/tools/sandbox are the default.
- Type hints everywhere; validate all model output before use.
- Every component independently testable; the platform runs end to end offline.

## 13. Definition of done

- `python -m kai.demo` runs offline: a sales-rep session applies role config + permissions,
  selects skills, loads only gated tools (+ pinned), runs sandboxed analysis over a filesystem,
  and produces a doc + dashboard artifact that updates on a follow-up turn.
- A test proves skill-gated loading (unselected tools never load) and that pinned skills always
  load.
- Two different surfaces call the same API and return identical results.
- `pytest` passes; swapping the stub harness/sandbox for real ones changes nothing about the
  platform interfaces.

## 14. Stretch goals

- Real agent harness (e.g. Deep Agents) + real isolated sandbox + warehouse/chat/docs connectors.
- Hybrid skill selection (retrieval/classifier prefilter → LLM final pick) for large catalogs.
- Per-user behavioral personalization (plan-first vs direct answers) learned automatically.
- Collaborative sessions: shared skills, shared artifacts, multi-user editing.
- Observability: per-session traces, token/cost dashboards, skill-gap detection from usage.
