# Context for continuing the insdi GSD briefs work

You are continuing work on the **insdi GSD briefs** project. This document gives you the full context you need to pick up where the previous session left off.

---

## Who I am and what I'm working on

I'm PJ, engineering owner of **insdi v2** — a multi-tenant B2B/B2C SaaS platform for collecting and managing form submissions. The platform has three service families:

- **Gather** — data collection (Templates, Links, Submissions)
- **Verify** — workflow orchestration (Flows, FlowRuns)
- **Calculate** — computation (Workbooks, Tests, CalculateRuns)

Plus a foundational **Platform** service (trust kernel — Orgs, Workspaces, AdminUsers, EndUsers, auth, federation). Architecture is on FastAPI + SQLAlchemy v2 + Pydantic v2 + Aurora Serverless v2 + DynamoDB + S3 on AWS, pinned to af-south-1 for HIPAA/POPIA/SOC2 compliance.

The platform design lives in a Notion Planning Hub. Key sections:
- §8.2 Service Boundaries — `https://www.notion.so/368c041b93b48167a404fb0b9c0db2c5`
- §8.3 Data Architecture & Storage — `https://www.notion.so/35ec041b93b4814f9ecdf32f5ee92d37`
- §8.4 API Surface (REST + MCP) — `https://www.notion.so/36bc041b93b481aa85dcde3a1db13d74`
- §8.6 Identity, Auth & Authorisation — `https://www.notion.so/35ac041b93b481169b79dd4734a63c05`

## What this project produces

A set of **GSD project briefs** that I'll paste into `/gsd-new-project` for each of four service Lambdas to scaffold their development:

1. `insdi-platform-service` (the trust kernel — must build first)
2. `insdi-gather-service`
3. `insdi-verify-service`
4. `insdi-calculate-service`

§8.2 specifies six Lambdas — we're deliberately scoping to four for now (Resources and Audit Lambdas are out of scope). Each service is a **separate GSD project** in its own repo.

The briefs are markdown files. They're not the actual code — they're the planning input for GSD.

## What GSD is

GSD (Get Shit Done) is a build methodology with commands like `/gsd-new-project`, `/gsd-discuss-phase`, `/gsd-plan-phase`, `/gsd-execute-phase`, `/gsd-verify-work`, `/gsd-ship`. It produces `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md` in each repo. The briefs feed it the vision and conventions on first run; GSD's own artefacts then become the working source of truth.

Original repo: `https://github.com/open-gsd/get-shit-done-redux` (formerly `gsd-build/get-shit-done`).

## The current state of the briefs

Eight files exist in a `gsd-briefs/` directory. The previous session produced them after several rounds of refinement. Layout:

```
gsd-briefs/
├── README.md                              ← index + how-to-use
├── _shared/
│   ├── INSDI_GSD_PRINCIPLES.md            ← layered-complexity methodology
│   ├── INSDI_API_CONVENTIONS.md           ← §8.4 distilled
│   └── INSDI_COMMONS_PROTOCOL.md          ← how to handle insdi-commons references
├── platform-service.md
├── gather-service.md
├── verify-service.md
└── calculate-service.md
```

Total ~2,850 lines.

## The five design decisions encoded in the briefs

These were grilled and locked through the previous session. They're the spine of everything:

### 1. Layered complexity — each milestone delivers a runnable tier

- **M1**: Bare CRUDLS, no auth, conventions correct. Runnable API.
- **M2**: Cognito JWT auth + request-level conventions (Idempotency-Key, ETag, rate limits).
- **M3**: Audit emission (PG outbox; the Audit Lambda drainer is out of scope, so events accumulate in PG).
- **M4**: Production hardening — RDS Proxy, Secrets Manager, af-south-1 deployment, MCP production architecture (per-audience server split, OAuth at session, `.well-known/mcp`, build-time manifest assembly, MCP session state in DynamoDB).
- **M5+**: Service-specific advanced features. **Drafting tools (`templates.draft_from_description` etc.) land M5 or M6+**, never earlier.

Each milestone produces a curl-able API AND an MCP-Inspector-exercisable tool surface. Never reach forward — a phase doesn't contain pieces of the next milestone's concern.

### 2. MCP parity from M1 P1 — surface convention, not complexity layer

insdi is AI-native. REST and MCP are "two surfaces over one state machine" (per §8.4 §1.4 / §9.3), structurally enforced by co-locating both in the same FastAPI Lambda. **Every REST route ships with its matching MCP tool in the same PR, from M1 P1.**

- Streamable HTTP transport, MCP server mounted at `/mcp` on the service Lambda's FastAPI app
- CRUDLS tool naming: `<resource>.{create, read, update, delete, list, search}` — note `read` (not `get`)
- Actions as `<resource>.<verb>` (e.g. `templates.publish`, `flow_runs.advance`, `tests.run`)
- Kebab-case URL → snake_case MCP namespace (`flow-runs` URL → `flow_runs` namespace)
- Both REST handlers and MCP tools are thin adapters calling the same use-case function in `services/`
- Same Pydantic models serve as REST request body AND MCP `inputSchema`
- Error envelope produced once; REST returns it as `application/problem+json`, MCP wraps it in `isError: true` content
- M1 auth placeholder: REST uses `X-Debug-Principal-Id` header, MCP uses `INSDI_DEBUG_PRINCIPAL_ID` env var
- Local testing: `npx @modelcontextprotocol/inspector http://localhost:<port>/mcp`

### 3. Typed-content discipline from M1 P1

**Substantive content fields use typed Pydantic models from `insdi-commons`. Never `dict[str, Any]`, never `Json`, never an untyped JSONB pass-through.**

This is what makes MCP's `inputSchema` carry the actual content rules (allowed field types as `Literal` enums, type-specific constraints via discriminated unions) so AI clients can produce valid input on the first try when an admin says e.g. *"create a patient intake form."* Without it, the AI sees `{"type": "object"}` and guesses.

Affected fields:
- `TemplateCreateInput.schema: commons.schemas.TemplateSchema` (Gather)
- `FlowCreateInput.definition: commons.schemas.FlowDefinition` (Verify)
- `WorkbookCreateInput.definition: commons.schemas.WorkbookDefinition` (Calculate)
- `Organisation.submitter_allowlist: commons.schemas.SubmitterAllowlist | None` etc. (Platform; used cross-service via policy chain)

Per my direction: most of these models are already defined in `insdi-commons` and should be referenced from M1 P1. If commons is incomplete, schedule a commons update — don't shadow locally with `dict`.

### 4. Drafting tools come later (M5 or M6+)

Natural-language → structured-input drafting tools (`templates.draft_from_description`, `flows.draft_from_description`, `workbooks.draft_from_description`, `organisations.draft_policies_from_description`) are M5+ work. They're backed by insdi-controlled LLM calls with curated prompts and few-shot examples. The admin describes intent, the tool proposes a structured input with rationale, the admin reviews before calling `.create`.

Why deferred: they require the structured surface to be proven, the typed content models to be stable, and a corpus of successful production inputs to draw few-shot examples from. Premature drafting tools produce inconsistent output that erodes trust in the platform.

### 5. The `insdi-commons` relationship

`insdi-commons` is a separate existing GSD project housing shared Pydantic models, ORMs, dependency functions, audit helpers, structured logger, etc. Not all of it exists yet. The protocol:

- When a service GSD project needs a cross-cutting symbol, it asks: "Can I assume `<symbol>` exists in commons, or implement locally first?"
- Local impls go in `src/<service_name>/_commons_pending/` with a header comment marking them temporary
- Import sites use the `_commons_pending` path, not `insdi_commons`, so the eventual migration is mechanical
- Tracked in `ROADMAP.md` under a "Migrate to insdi-commons" section
- Migration is its own phase, never bundled with feature work

## What's in each brief

Each service brief covers:
- Vision (for PROJECT.md)
- Entities owned per §8.2
- Milestone roadmap with phase sketches for M1
- Decisions to grill during `/gsd-discuss-phase 1` (and later milestones)
- Out-of-scope-for-M1 lists (deferrals are explicit)
- URL surface tables (M1 and M5+)
- MCP tool surface tables
- M1 acceptance criteria with both curl (REST) and MCP Inspector sections
- Things to watch / known tensions

Platform-service has the most detail because it's the trust kernel. Gather-service has a substantive "Typed-content discipline for AI usability" section because that's where the typed-content rule lands most visibly (Template `schema`).

## How to use the briefs

For each service repo:

1. Run `/gsd-new-project` in the empty (or scaffold-only) repo
2. When GSD asks for context, paste in (in this order):
   - `_shared/INSDI_GSD_PRINCIPLES.md`
   - `_shared/INSDI_API_CONVENTIONS.md`
   - `_shared/INSDI_COMMONS_PROTOCOL.md`
   - The service-specific brief (e.g. `platform-service.md`)
3. GSD produces `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md`
4. Run `/gsd-discuss-phase 1` — the brief lists specific decisions to grill on
5. `/gsd-plan-phase 1`, `/gsd-execute-phase 1`, `/gsd-verify-work 1`, `/gsd-ship 1`
6. Repeat for each phase, milestone

Recommended build order: **platform-service first** (Org/Workspace ORMs are read by the others), then the other three can be built in parallel or sequentially.

## My working style (please honour these)

- **Decisions get grilled before being locked.** Don't pick options silently — surface them, give pros/cons, wait for me to choose. Concise pros/cons, then ask.
- **No writing to Notion until full agreement is reached.** Drafting can happen locally; publishing is a separate explicit step.
- **Sequential progression.** Flag when steps are being skipped. Verified step-by-step beats jumping ahead.
- **Push back when grounded.** I accept well-reasoned technical pushback. Don't just agree.
- **Project rules belong in Claude instructions, not inside the documents themselves.**
- **The `/grill-me` pattern**: when I ask to be grilled on a plan or design, challenge my assumptions and force precise decisions before anything is locked.
- **First-principles planning.** No PRD-era contamination — insdi v2 is greenfield.

## What you'd typically be asked to do next

Plausible next steps (not all of them apply — wait for me to direct):

- **Set up the repos.** Create the four service repos, drop in the briefs (with the shared files via symlink or sync, not copy — see README), kick off `/gsd-new-project` for platform-service first.
- **Refine a brief based on something that came up.** E.g. a constraint I didn't see, a §8.x decision that changed, a missed cross-service dependency.
- **Walk through `/gsd-discuss-phase 1` for platform-service.** The brief lists the questions; I'd grill on each.
- **Plan the `_commons_pending/` convention concretely** before P1 starts — header comment exact text, directory structure, import path conventions.
- **Pick the MCP server library** (`fastapi-mcp` vs `mcp-server-fastmcp` vs hand-rolled). This is locked once for all four services.
- **Resolve open §8.4 questions** flagged in the briefs (e.g. idempotency table substrate, ID-prefix conventions).
- **Continue refining the briefs themselves** if I spot something else that's wrong or missing.

## Files I want you to read first

Before doing anything else, view the existing brief files. Their layout:

```
gsd-briefs/README.md
gsd-briefs/_shared/INSDI_GSD_PRINCIPLES.md
gsd-briefs/_shared/INSDI_API_CONVENTIONS.md
gsd-briefs/_shared/INSDI_COMMONS_PROTOCOL.md
gsd-briefs/platform-service.md
gsd-briefs/gather-service.md
gsd-briefs/verify-service.md
gsd-briefs/calculate-service.md
```

If they're not in the working directory, ask me where they are. They were produced in a prior Claude session; I have them saved.

## What to do right now

Acknowledge you've read this context. Then ask me what I want to work on, with a few suggestions based on the "next steps" list above. Don't start writing or modifying files until I direct you.
