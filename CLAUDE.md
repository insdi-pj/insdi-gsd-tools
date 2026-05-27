# CLAUDE.md — working context for the `gsd-prompts` repo

This repo holds **GSD project briefs** for the insdi v2 platform — the planning context pasted
into `/gsd-new-project` for four service Lambdas. It does **not** hold service code; the briefs
are the planning *input*. Source-of-truth for the methodology lives in `_share/` (read those for
detail; this file is orientation + rules).

> Derived from `CLAUDE_CODE_CONTEXT.md` (the session-handoff doc). When the two disagree, the
> live files in `_share/` and the four `*-service.md` briefs win.

## What this produces

Four briefs, each pasted into `/gsd-new-project` in its own dedicated repo:

1. `insdi-platform-service` — trust kernel (Orgs, Workspaces, AdminUsers, EndUsers, auth). **Build first** — the other three read its Org/Workspace ORMs via direct SQL on the shared `public` schema.
2. `insdi-gather-service` — Templates, Links, Submissions.
3. `insdi-verify-service` — Flows, FlowSteps, FlowRuns.
4. `insdi-calculate-service` — Workbooks, Sets, Tests, CalculateRuns.

§8.2 specifies six Lambdas; **Resources and Audit are deliberately out of scope** (four-service scope).
Platform/stack: FastAPI · SQLAlchemy v2 · Pydantic v2 · Aurora Serverless v2 · DynamoDB · S3, pinned
to `af-south-1` for HIPAA/POPIA/SOC2. Design lives in a Notion Planning Hub (§8.2 Service Boundaries,
§8.3 Data Architecture, §8.4 API Surface REST+MCP, §8.6 Identity/Auth).

## Repo layout (actual)

```
gsd-prompts/
├── CLAUDE.md                  ← this file
├── CLAUDE_CODE_CONTEXT.md     ← original session-handoff doc
├── README.md                  ← index + how-to-use
├── _shared/                   ← CANONICAL shared docs (MCP-enabled rewrite)
│   ├── INSDI_GSD_PRINCIPLES.md
│   ├── INSDI_API_CONVENTIONS.md
│   └── INSDI_COMMONS_PROTOCOL.md
├── _shared_old/               ← pre-MCP backup of the shared docs (NOT canonical)
├── platform-service.md
├── gather-service.md
├── verify-service.md
└── calculate-service.md
```

The canonical shared folder is `_shared/` — it matches every doc reference. (`_shared_old/` is the
pre-MCP backup; not canonical.) Note: the handoff doc `CLAUDE_CODE_CONTEXT.md` describes a
`gsd-briefs/` parent dir that does not exist — files are at repo root.

## The five locked decisions (spine — full detail in `_share/INSDI_GSD_PRINCIPLES.md`)

1. **Layered complexity.** M1 bare CRUDLS / no auth → M2 Cognito auth + request conventions
   (Idempotency-Key, ETag, rate limits) → M3 audit emission (PG outbox; no drainer in four-service
   scope, events accumulate in PG) → M4 prod hardening + MCP production architecture → M5+
   service-specific advanced features. **Never reach forward** — a phase contains no pieces of a later
   milestone's concern. Each milestone is runnable: curl-able REST **and** MCP-Inspector-exercisable.
2. **MCP parity from M1 P1** (surface convention, not a complexity layer). Every REST route ships its
   matching MCP tool in the **same PR**; both are thin adapters over one `services/` use-case function;
   CRUDLS naming `<resource>.{create,read,update,delete,list,search}` — `read`, not `get`; actions as
   `<resource>.<verb>`; kebab-case URL → snake_case MCP namespace. Same Pydantic model is REST body
   AND MCP `inputSchema`. M1 auth placeholder: REST `X-Debug-Principal-Id` header, MCP
   `INSDI_DEBUG_PRINCIPAL_ID` env var.
3. **Typed-content discipline from M1 P1.** Substantive content fields use typed Pydantic models from
   `insdi-commons` — never `dict[str, Any]`, `Json`, or untyped JSONB. (`Template.schema`,
   `Flow.definition`, `Workbook.definition`, `submitter_allowlist`, etc.) This is what makes the MCP
   `inputSchema` genuinely AI-usable. Carve-outs (audit payloads, per-Workbook run outputs) are
   explicitly justified.
4. **Drafting tools come later (M5/M6+).** `*.draft_from_description` tools are LLM-backed and need the
   structured surface proven, typed models stable, and a corpus for few-shot examples. Never M1.
5. **`insdi-commons` protocol.** Ask before assuming a commons symbol exists. Local fallbacks live in
   `src/<service>/_commons_pending/` with the standard header, imported via the `_commons_pending`
   path, tracked in `ROADMAP.md` under "Migrate to insdi-commons". Migration is its own phase.

## Skill conventions (in `_shared/INSDI_GSD_PRINCIPLES.md` §10)

Four named skills are wired into the methodology as **instruments, not authors** — they supply rigour/
knowledge to accomplish a goal the principles + §8.4 + the brief already own; they must not rescope or
redesign. On any conflict between a skill default and an insdi convention, **the insdi convention wins**
(note the divergence and surface it).

- **`/grill-me`** (§8) — resolve a genuinely open decision to a precise, locked answer. This doc, the
  brief, and `/gsd-discuss-phase` decide *what* gets grilled; the skill drives it to an answer + rationale
  (both recorded in `CONTEXT.md`). Must not broaden/redirect the question — if the real decision turns out
  different or larger, stop and surface a scoping change to PJ.
- **`/mcp-builder`** (§3.7) — build the MCP surface. Supplies transport setup, schema/tool-description
  generation, Inspector testing, spec correctness. Design is fixed by §3.1–§3.6 / §8.4 §9 / the brief
  (co-location at `/mcp`, streamable HTTP, CRUDLS `read`-not-`get`, `noun.verb`, kebab→snake, use-case
  delegation, shared Pydantic models, M1→M4 milestone shape).
- **`/generate-tests`** (§10.1) — write a phase's tests. Many args; **one invocation per test layer**:
  `--type=contract` for `schemas/` (covers typed-content §4), `--type=unit` for `services/` + helpers,
  `--type=integration` for `api/` + `mcp/` (incl. MCP-parity round-trips). `--depth` scales with the
  milestone; `--mock=external` default, `--mock=none` for end-to-end against local PG/DDB. NB: `--type`
  is `unit|integration|contract` — there is **no** `e2e` value; e2e = the brief's full-lifecycle
  acceptance scripts, run as `--type=integration --mock=none`.
- **`/generate-docs`** (§10.2) — **inline** docstrings/comments in source only. It does **not** create
  separate doc files: `PROJECT.md`/`REQUIREMENTS.md`/`ROADMAP.md`/`README.md` are GSD / `gsd-docs-update`
  territory, not `/generate-docs` output. MCP tool descriptions (in the `@mcp_tool` decorator) are code,
  so they're fair game.

These skills live at user level (`~/.claude/skills/`) and in the service repos' `.claude/`, so they're
available where GSD consumes these docs. `/generate-docs`, `/grill-me`, and `/mcp-builder` are not
auto-surfaced (invoke explicitly / confirm they're installed in the target repo before relying on them).

## Working style — honour these (PJ)

- **Grill before locking.** Surface decisions with concise pros/cons and wait for PJ to choose — never pick silently.
- **No writing to Notion until full agreement.** Drafting locally is fine; publishing is a separate explicit step.
- **Sequential progression.** Flag when steps are being skipped; verified step-by-step beats jumping ahead.
- **Push back when grounded.** Well-reasoned technical pushback is welcome — don't just agree.
- **Project rules belong in Claude instructions (this file), not inside the briefs themselves.**
- **First-principles, greenfield.** insdi v2 has no PRD-era baggage.
- **Don't modify files until directed.** Acknowledge, surface options, wait.

## How the briefs are used

Per service repo: `/gsd-new-project`, pasting (in order) `_share/INSDI_GSD_PRINCIPLES.md`,
`_share/INSDI_API_CONVENTIONS.md`, `_share/INSDI_COMMONS_PROTOCOL.md`, then the service brief.
GSD produces `PROJECT.md`/`REQUIREMENTS.md`/`ROADMAP.md`; then `/gsd-discuss-phase 1` (the brief lists
decisions to grill), `/gsd-plan-phase 1`, `/gsd-execute-phase 1`, `/gsd-verify-work 1`, `/gsd-ship 1`.
Build order: **platform-service M1 first**, then the other three in parallel or sequentially.
`/gsd-verify-work` enforces §8.4 conventions, MCP parity, and typed-content discipline.

## Open foundational decisions (not yet locked)

- MCP server library — `fastapi-mcp` vs `fastmcp` vs hand-rolled. **Locked once for all four services.**
- `_commons_pending/` convention specifics — exact header text, sub-structure, import paths.
- Idempotency table substrate (Postgres vs DynamoDB — §8.4 open question), ID-prefix conventions.
