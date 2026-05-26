# insdi GSD Project Principles

**Status:** Authoritative process doc — applies to every GSD project for an insdi service Lambda
**Audience:** GSD (`/gsd-new-project`, `/gsd-discuss-phase`, `/gsd-plan-phase`, `/gsd-execute-phase`) when planning and executing work on insdi services
**Owner:** PJ (engineering)

This document defines **how GSD should structure work** on any insdi service Lambda. It is not a service spec — it is the meta-process every service follows. The service-specific brief (e.g. `platform-service.md`) describes *what* a service does; this document describes *how the project is shaped over time*.

Read this before drafting `PROJECT.md`, `REQUIREMENTS.md`, or `ROADMAP.md`. Reread when starting a new milestone.

---

## 1. Why layered complexity

The single most important rule:

> **A milestone delivers a runnable, testable API. The next milestone adds one tier of complexity on top. Never write a feature at production complexity before its precursors exist.**

This mirrors how a human engineering team builds: get the route working, then add auth, then add audit, then harden. The reason matters — it is not just aesthetic preference:

- **When something goes wrong, you fix one tier.** If auth is wrong in M2, you debug auth code. You don't refactor through three other concerns that were piled in at the same time.
- **Each milestone is a real demo.** M1 produces an API a human can curl. M2 produces an API a real client can sign into. M3 produces an audit trail. Each is shippable in the sense of being demonstrable to a stakeholder.
- **Decisions get tested cheaply.** If the resource shape is wrong, M1 reveals it before auth is built on top.
- **GSD's parallel execution stays bounded.** Subagents have a much easier time building "the create-org endpoint with no auth" than "the create-org endpoint with Cognito JWT validation, AdminUser hydration, RLS, audit emission, and rate-limiting" all at once.

If a phase plan starts pulling in concerns from a later milestone "because it'll need them eventually" — **stop the phase, refuse the plan, and re-scope.** Premature integration is the failure mode this discipline exists to prevent.

---

## 2. The milestone structure

Every insdi service GSD project follows the same milestone shape. The exact phase list inside each milestone depends on the service, but the **tier each milestone delivers** is fixed.

### M1 — Bare functional API, conventions-correct, no auth

**Tier delivered:** A runnable FastAPI Lambda with basic CRUD against its owned entities, following §8.4 conventions for URL structure, response shape, error envelope, naming, and pagination — but with **no authentication layer**.

**What "no auth" means:**
- Routes are still under `/v1/{namespace}/...` per §8.4 §3 — URL conventions are not optional, ever
- Principal identity is injected via a development-only header `X-Debug-Principal-Id` (header name is the same across all four services) — this header would never exist in production, but in M1 it acts as the placeholder for whatever Cognito-derived `UserContext` will eventually carry
- No Cognito user pools provisioned, no JWT validation, no JWKS, no `RequireAdmin`/`RequireEndUser` dependencies
- No Org-membership check, no Workspace-membership check, no RLS — just "if the header has a value, use it"
- No agent grants, no federation, no link-scoped JWTs, no OAuth endpoints

**What is required and non-negotiable in M1:**
- URL versioning (`/v1/...`), audience namespaces (`/v1/admin/...`, etc.) per §8.4 §3.2
- Resource paths: plural-noun kebab-case, domain vocabulary not service vocabulary (`templates`, not `gather-templates`) per §8.4 §1.3
- Bare-object responses for single resources, `{ items, page }` envelope for lists, per §8.4 §5.1 and §4.1
- RFC 9457 `application/problem+json` error envelope with `code` and `request_id` extensions per §8.4 §5.3
- Error code taxonomy: `auth.*`, `validation.*`, `policy.*`, `conflict.*`, `not_found.*`, `rate_limit.*`, `internal.*` per §8.4 §5.4
- HTTP status mapping per §8.4 §5.5 (in particular: 404 conflates "doesn't exist" with "not yours")
- Cursor pagination with opaque cursors per §8.4 §4.1; no `total` count anywhere
- Three-schema Pydantic pattern (Create, Update, Response) per insdi convention; `model_validate()` with `from_attributes=True` for ORM → response conversion
- SQLAlchemy v2 ORM, Pydantic v2 schemas, Alembic migrations, FastAPI
- Structured JSON logging to stdout per §8.2 §8.5

**What is explicitly deferred:**
- Cognito and JWT validation → M2
- `Idempotency-Key` enforcement → M2 (header may be accepted but ignored in M1; if accepted, the validation message in the OpenAPI spec should mark it as "ignored in M1")
- `ETag` / `If-Match` enforcement → M2
- Rate limiting → M2
- `_deprecation` envelope → not relevant until M2+ (no deprecated versions yet)
- Audit emission → M3
- MCP surface → deferred (later milestone)
- Federation, agent grants, link-scoped JWTs → later
- RDS Proxy → M4+ (M1/M2/M3 use direct Aurora connection or local Postgres)

### M2 — Authentication and request-level conventions

**Tier delivered:** The M1 API now requires real authentication. Cognito JWT validation, `RequireAdmin`/`RequireEndUser` dependencies, `UserContext` construction, Org membership checks. Idempotency, ETag, rate-limit headers are now real.

**What M2 adds:**
- Cognito user pools (admin + end-user per §8.6 §1.3) — provisioned via the `insdi-infra` CDK repo or a stub equivalent
- `RequireAuth` / `RequireAdmin` / `RequireEndUser` / `RequireSubmissionSession` / `RequireAdminOrAgent` dependencies per §8.2 §6
- `UserContext` family (AdminUserContext, EndUserContext, SubmissionSessionContext, AgentContext placeholder) per §8.2 §6
- The `X-Debug-Principal-Id` header is removed; principals now come from validated JWTs
- DB hydration of UserContext (Memberships, WorkspaceMemberships)
- Per-route auth dependency declarations on every existing route
- `Idempotency-Key` enforcement per §8.4 §6.1–6.3 (24h retention, partitioned by `org_id`)
- `ETag` on single-resource GETs; `If-Match` required on PATCH/PUT and on DELETE of high-value resources per §8.4 §6.4–6.6
- Rate-limit response headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-RateLimit-Bucket`) per §8.4 §7.5; per-Org and per-IP buckets, DynamoDB-backed
- `Sunset` / `Deprecation` header plumbing (no value yet — v1 isn't deprecated — but the middleware that emits them must exist)

**What is still deferred:**
- Audit emission (still M3)
- MCP surface (still later)
- Federation, agent grants (later)
- RLS (deferred until per-service schemas, §8.2 §1.4)

### M3 — Audit emission and observability

**Tier delivered:** Every write emits an audit event per §8.2 §5. Tier C reads emit operational telemetry. Request IDs propagate end-to-end.

**What M3 adds:**
- `audit.emit_with_tx()` for transactional audits (Tier A/B writes coupled to PG changes)
- `audit.emit_standalone()` for non-transactional audits (sign-ins, token operations, errors)
- Direct DDB writes for Tier C operational telemetry
- PG outbox tables: `audit_outbox_transactional` and `audit_outbox_standalone` per §8.2 §5.1–5.2
- `X-Request-Id` header set on every response; correlated in logs, audit events, error envelopes
- Structured exception logging per §8.2 §8.5

**Important gap to flag:** §8.2 specifies a separate `insdi-audit-service` Lambda that drains the outbox tables into DynamoDB. The four-service scope does not include this Lambda. In M3, the outbox tables are written but **not drained** — audit events accumulate durably in PG until an audit-service Lambda exists. This is acceptable for development and is the same failure mode §8.2 §9 lists ("Audit Lambda down: outbox rows accumulate in PG (durable). Drain catches up on recovery."). Flag this in `ROADMAP.md` for the eventual audit-service work.

### M4 — Production hardening

**Tier delivered:** RDS Proxy, KMS, observability hooks, OpenAPI assembly pipeline contributions, deployment to staging.

**What M4 adds:**
- RDS Proxy in front of Aurora per `insdi-platform-service`'s memory and §8.3
- Secrets Manager integration for DB creds, JWT signing keys
- Per-region deployment with `af-south-1` pinning per insdi compliance posture
- `/openapi.json` and `/mcp-fragment.json` outputs ready for the deploy-time assembly pipeline (§8.4 §11)
- Lambda cold-start optimisation; cold-start log emission per §8.2 §8.5
- SAM template tuned to production shape
- Pydantic BaseSettings for config; env vars now, Parameter Store later

### M5+ — Service-specific advanced features

Each service has its own M5+ shape — federation, agent grants, link-scoped JWTs, MCP surface, advanced policy resolution, file uploads, flow orchestration, computation engines. These are spelled out in each service's brief.

---

## 3. Phase shape within a milestone

Inside each milestone, phases follow the same principle one level down: **each phase ships one runnable slice**, not one runnable feature-with-no-tests. The slice has:

- One coherent capability (typically one entity or one workflow)
- All §8.4 conventions correctly applied to that capability
- Tests passing for that capability
- Existing capabilities still work

A reasonable phase size: one entity's CRUD + a meaningful subset of its routes, OR one cross-cutting concern (e.g. "wire Cognito JWT validation on existing routes") applied across the service.

**A phase is too big if** it cannot be planned in one `/gsd-plan-phase` pass, or if its execution would touch more than ~6-8 files significantly. Split it.

**A phase is too small if** the deliverable is a single function with no observable effect on the API. Combine with the next.

---

## 4. The `insdi-commons` relationship

`insdi-commons` is a separate existing GSD project hosting Pydantic models, SQLAlchemy ORMs, FastAPI dependency functions, audit helpers, the structured logger, link-scoped JWT issuance, the event-schema registry, and other shared code per §8.2 §1.1.

**Not all of insdi-commons exists yet.** Many of the things a service GSD project will need — `RequireAdmin`, `audit.emit_with_tx`, `commons.logging.get_logger`, specific ORMs — may or may not be available when the service project starts.

**The protocol for any service GSD project (planning phase, execution phase, or verify phase):**

1. When the project needs a model, ORM table, dependency function, helper, or constant that **should logically live in `insdi-commons`**:
   - State explicitly what it needs and what shape it should have (signature, returned type, behaviour)
   - Ask the user: *"Can I assume `<symbol>` exists in `insdi-commons` with this shape, or should I implement it locally first?"*
2. If the user confirms it's in commons, GSD uses it via normal import.
3. If the user says "implement locally":
   - Implement it inside the service repo, in a clearly-marked `_commons_pending/` subpackage (or equivalent — establish the convention in the service's M1 P1)
   - Add a file-level comment: `# TODO: Move to insdi-commons (see ROADMAP)`
   - Add a `TODO:` to `ROADMAP.md` under a "Migrate to insdi-commons" section, with the symbol name, the path it lives at locally, and a one-line description of what insdi-commons needs to provide
   - When the symbol later lands in insdi-commons, the migration is the deletion of the local copy and a swap to the import — a separate phase, not bundled with feature work
4. Never invent an interface that contradicts what's documented in §8.2 §1.2 or §8.4. Local implementations match the canonical shape; the move-to-commons step is then a relocation, not a redesign.

**Examples of things commonly needed:**

- ORMs for AdminUser, EndUser, Organisation, Workspace, Membership (Platform-owned per §8.2 §2) — Gather/Verify/Calculate read these via direct SQL on the shared `public` schema, so they need the ORMs to read with
- `EntityRef` Pydantic model used in audit emission targets
- `commons.logging.get_logger(__name__)`
- `commons.audit.emit_*` helpers
- `commons.platform_auth.issue_link_scoped_jwt()` (only relevant once link-scoped JWTs come in, later milestone)
- The cursor-pagination helper that encodes/decodes opaque cursors and validates them against the sort+filter state
- The error-envelope builder that produces RFC 9457 problem+json responses
- The operator-suffix filter parser

The first three are likely needed in M1. The auth-related ones come in at M2. The link-scoped JWT in M5+.

---

## 5. The §8.4 reference protocol

§8.4 (API Surface — REST + MCP) at `https://www.notion.so/36bc041b93b481aa85dcde3a1db13d74` is the **single source of truth** for API conventions. Every service follows it.

The service brief inlines a distilled checklist of §8.4 rules that apply at each milestone (in the brief itself). But:

- For any case that isn't crystal-clear in the brief, GSD **fetches §8.4 directly** during research or planning rather than inferring.
- The shared `INSDI_API_CONVENTIONS.md` in this directory captures the most-referenced rules as a quick lookup; it does not replace §8.4.
- If §8.4 is updated between milestones, the change may invalidate previous milestone work — flag it.

Other §8.x pages worth knowing about:

- §8.2 Service Boundaries — `https://www.notion.so/368c041b93b48167a404fb0b9c0db2c5` — entity ownership, request lifecycle, audit emission, UserContext
- §8.3 Data Architecture & Storage — `https://www.notion.so/35ec041b93b4814f9ecdf32f5ee92d37` — three-store design, partitioning, schema layout
- §8.6 Identity, Auth & Authorisation — `https://www.notion.so/35ac041b93b481169b79dd4734a63c05` — Cognito pools, federation, agent grants, link-scoped JWTs
- Entities and Relationships — `https://www.notion.so/33dc041b93b480e98976f0280c5872dd` — the canonical entity diagram

---

## 6. When in doubt — grill, don't guess

PJ's working style: decisions are grilled before being locked. This applies to GSD too.

If a phase plan involves a decision that's not explicitly settled in the service brief or in §8.4 — for example:

- "Should the policy-resolution endpoint live on the Org or as a standalone resource?"
- "Should `submitter_allowlist` be a JSON column or a separate table?"
- "What's the cursor encoding for the case where the sort key is a JSON path?"

— **do not just pick one in the plan.** Surface the question during `/gsd-discuss-phase` and have the user resolve it. The discuss step exists for exactly this purpose; use it.

Once a decision is locked in `CONTEXT.md`, downstream phases follow it. If a later decision contradicts an earlier one, surface the conflict explicitly — don't quietly let one win.

---

## 7. Convention compliance is part of `/gsd-verify-work`

`/gsd-verify-work` for any insdi service must verify:

- URL structure follows §8.4 §3
- Response shapes follow §8.4 §5
- Error envelopes follow §8.4 §5.3–5.4
- Pagination follows §8.4 §4.1
- Status codes follow §8.4 §5.5
- Naming follows §8.4 §1.3 (kebab-case, plural-noun, domain vocabulary)
- The milestone's deferred concerns are *not present yet* (e.g. M1 work should not contain JWT validation code; M2 should not contain audit emission code)

A verify step that catches "this is correct but for a later milestone" is doing its job — that work either gets moved or the milestone scope gets adjusted, but it doesn't ship muddled.

---

## 8. Summary — the four rules

1. **Convention before complexity.** §8.4 conventions are non-negotiable from M1 P1. Complexity layers in over milestones.
2. **Each milestone is runnable.** A milestone that doesn't produce a curl-able API on completion is not done.
3. **Don't reach forward.** A phase doesn't contain pieces of the next milestone's concern.
4. **insdi-commons is the source of truth for shared code.** Local implementations are temporary and tracked.
