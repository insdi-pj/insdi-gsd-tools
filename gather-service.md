# GSD Project Brief — `insdi-gather-service`

**Lambda:** `insdi-gather-service`
**Domain:** Data collection — Templates, Links, Submissions, in-flight submission journey, finalisation
**Repo:** dedicated repo (separate GSD project)
**Status:** Brief for `/gsd-new-project`

**Read first:**
- `_shared/INSDI_GSD_PRINCIPLES.md` — the milestone/phase methodology
- `_shared/INSDI_API_CONVENTIONS.md` — §8.4 distilled
- `_shared/INSDI_COMMONS_PROTOCOL.md` — how to handle commons references

**Reference (fetch as needed during research/planning):**
- §8.2 Service Boundaries — `https://www.notion.so/368c041b93b48167a404fb0b9c0db2c5`
- §8.3 Data Architecture & Storage — `https://www.notion.so/35ec041b93b4814f9ecdf32f5ee92d37` (Gather has substantial DDB interaction in M5+)
- §8.4 API Surface — REST + MCP — `https://www.notion.so/36bc041b93b481aa85dcde3a1db13d74`
- §8.6 Identity, Auth & Authorisation — `https://www.notion.so/35ac041b93b481169b79dd4734a63c05` (§4 policy chain is core Gather work)
- The Identity, Auth & Authorisation design (auth doc) — in project files; §4 (auth-floor + allowlist policy chain) is essential reading

---

## 1. Vision (for `PROJECT.md`)

`insdi-gather-service` is the heart of the data-collection product. It owns **Templates** (versioned definitions of what to collect), **Links** (concrete shareable URLs that point at a Template configured for a specific audience), and **Submissions** (the actual collected data). It also owns the **in-flight submission journey** — the stateful process a submitter goes through filling out a Template via a Link, persisted in DynamoDB until finalisation flips the canonical row into Postgres.

Gather is the service that downstream services (Verify, Calculate, Resources via Triggers) react to. Almost every cross-service workflow in insdi starts with a `submission.finalised` event from Gather (§8.2 §3.7).

Gather's central nuance: the **policy resolution pattern**. Templates and Links store *raw declared config* for `auth_required_floor` and `submitter_allowlist`, but their API responses carry *computed effective values* resolved by walking the `Organisation → Workspace → Template → Link` chain per §8.6 §4 / the auth doc §4. This is the single most important architecture pattern in the service. Get it right in M1 and it threads through everything; get it wrong and refactoring is painful.

In v1 scope: Template/Link/Submission CRUD over REST. The Gather Template policy chain. Submission session lifecycle. File attachments (deferred until Resources exists, but referenced in the data model).

---

## 2. Entities owned (from §8.2 §2)

| Entity | Storage | Notes |
|---|---|---|
| `Template` (versioned) | PG | The form definition; versioned (immutable past versions). Stores raw `auth_required_floor`, `submitter_allowlist` |
| `Link` | PG | A shareable URL pointing at a specific Template version with optional overriding policy fields. Stores raw `auth_required_floor`, `submitter_allowlist` |
| `Submission` (finalised) | PG | The canonical row, written on finalise. Contains the validated body |
| Submission journey events (in-flight) | DDB | The stateful in-progress submission; one `STATE#CURRENT` + many `EVENT#*` per session per §8.3 |
| Submission session (anonymous) | DDB | Session metadata + revocation/expiry state |

**Important cross-service reads (via direct SQL on shared `public` schema, no HTTP):**

- `Organisation` (Platform-owned) — for the policy chain
- `Workspace` (Platform-owned) — for the policy chain and ownership context
- `Membership`, `WorkspaceMembership` (Platform-owned) — for AdminUser permission checks in admin routes
- `EndUser` (Platform-owned) — for authenticated submitters
- `File` (Resources-owned) — for file attachments **but Resources is out of scope**; see §8 below

---

## 3. Milestone roadmap

### M1 — Bare CRUD, no auth, conventions-correct

**Goal:** A FastAPI Lambda where a human can create a Template, configure a Link to it, simulate a submission session against the Link, and finalise. All under correct §8.4 conventions. No authentication.

A human can:
- Create a Template (with a basic JSON schema describing fields)
- Create a Link pointing at the Template
- Start a submission session against the Link
- Write field-changes during the session
- Finalise the session — turning it into a Submission row
- List/search Templates, Links, Submissions

**Phases (sketch — confirm in `/gsd-discuss-phase 1`):**

- **P1 — Scaffold:** Same shape as platform-service P1 — repo layout, SAM, FastAPI, Postgres locally, Alembic, structured logger, error envelope, pagination, filter parser, `_commons_pending/` convention. Also: **local DynamoDB** (DynamoDB Local in Docker or moto), since Gather uses DDB for in-flight state from M1.
- **P2 — Cross-service ORMs (read-only):** Define the SQLAlchemy ORMs Gather needs to *read* from Platform-owned tables: `Organisation`, `Workspace`. Add to `_commons_pending/models/` (ask the user — these ORMs really belong in commons, see protocol doc). No migrations for these — Gather doesn't own them, doesn't migrate them. In a real shared-schema deployment, Platform owns the migrations. **For M1 development, run platform-service's migrations first to set up the schema, then Gather can read the tables.**
- **P3 — Template CRUD (basic):** Three-schema Pydantic. Template ORM with `template_id`, `workspace_id`, `name`, `description`, `version`, `schema` (JSONB — the field definitions), `auth_required_floor`, `submitter_allowlist`, `created_at`, `updated_at`. **No versioning logic in M1** — just store a version number, default 1. M5+ adds publish-creates-new-version behaviour. Routes: `POST /v1/admin/templates`, `GET /v1/admin/templates`, `POST /v1/admin/templates/search`, `GET /v1/admin/templates/{tpl_id}`, `PATCH /v1/admin/templates/{tpl_id}`, `DELETE /v1/admin/templates/{tpl_id}`.
- **P4 — The policy chain:** Implement `resolve_effective_policy(template_id)` that walks `Template → Workspace → Organisation` and computes `effective_auth_floor` and `effective_allowlist` per §8.6 §4. This is a use-case function (in `services/`). Used by the Template GET/PATCH responses to populate the computed fields. **Implement the inheritance rules correctly from day one** — nearest-ancestor-wins, tighten-only-downward, explicit refusal on widen attempts. This pattern repeats for Link in P5; getting it solid here saves rework.
- **P5 — Link CRUD:** Link ORM with `link_id`, `template_id`, `short_id` (URL-friendly), `auth_required_floor`, `submitter_allowlist`, plus the effective fields computed by walking `Link → Template → Workspace → Organisation`. Routes: `POST /v1/admin/links`, `GET /v1/admin/links`, `POST /v1/admin/links/search`, `GET /v1/admin/links/{link_id}`, `PATCH /v1/admin/links/{link_id}`, `DELETE /v1/admin/links/{link_id}`. Validate at create/update time that the Link's declared policy does not widen its parent — reject with `422 policy.allowlist_widens_inherited` or `422 policy.auth_floor_conflict`.
- **P6 — Submission session start:** `POST /v1/public/submission-sessions` accepts a Link short_id, walks the policy chain, generates a session UUID, writes the session metadata to DDB (one item: `PK = SESSION#{id}`, `SK = META`). Returns the session context to the client. In M1 this works without any auth on the public side — the session ID + session cookie semantics from §8.2 §4.2 are simulated; an `X-Debug-Submission-Session-Id` header (analogous to `X-Debug-Principal-Id`) carries the session for subsequent calls.
- **P7 — Field changes:** `POST /v1/public/submission-sessions/{id}/field-changes` writes one `EVENT#{event_id}` item and updates `STATE#CURRENT` in DDB. Per §8.3, one `TransactWriteItems` per change. **Simplify in M1:** skip the TransactWriteItems pattern; just do two separate writes for now (note this as a known limitation). M3 or M4 corrects to atomic.
- **P8 — Finalise:** `POST /v1/public/submission-sessions/{id}/finalise` reads `STATE#CURRENT`, validates against the Template's schema, inserts canonical row into PG `submissions`, marks DDB `META` as `submitted`. Returns the finalised Submission. **No EventBridge publication in M1** — `submission.finalised` event is added later.
- **P9 — Submission CRUD (admin-side):** `GET /v1/admin/submissions`, `POST /v1/admin/submissions/search`, `GET /v1/admin/submissions/{sub_id}`. DELETE deliberately deferred — submissions don't get hard-deleted; erasure flow comes later.
- **P10 — Polish:** Manual test script that exercises the full lifecycle (create Template → create Link → start session → field changes → finalise → read submission). OpenAPI sanity check.

**Decisions to grill in `/gsd-discuss-phase 1`:**
- The Template `schema` field shape — JSON Schema? a custom shape? what field types are supported in M1 (text, number, date, choice — what about file, multi-choice, signature)?
- Link `short_id` generation — short hash? human-readable? collision strategy?
- Submission body shape — `{ "field_id": value, ... }` keyed by Template field IDs? schema-validated server-side at finalise?
- Whether Submission has a `status` enum and if so what values M1 needs (`finalised` is enough; `processing` and others come later)
- DDB table shape for sessions — confirm `PK = SESSION#{id}`, `SK = META | STATE#CURRENT | EVENT#{event_id}` aligns with §8.3
- Whether the policy chain refusal returns 422 with a structured `errors` array showing which level set what (recommended) or just a single error
- The Workspaces/Memberships read pattern — since Gather doesn't own them, the M1 SQL reads happen against tables created by Platform's M1 migrations. If platform-service M1 is not yet shipped, decide whether to stub the Workspace table here in Gather to unblock M1 work (and migrate to Platform-owned later) or pause Gather M1 until Platform M1 P3 is done

**Out of scope for M1:**
- JWT validation, real auth (M2)
- Audit emission (M3)
- Idempotency, ETag, rate-limit (M2)
- EventBridge publication of `submission.finalised` and others (M5+)
- File attachments (M5+ once Resources exists)
- Template versioning / publish flow (M5+)
- Trigger evaluation (Resources-owned; out of scope here)
- Link-scoped JWT issuance for verified-identity submissions (M5+ in Platform)
- Verify Flow integration (M5+)
- Calculate Workbook integration (M5+)

---

### M2 — Cognito JWT validation, real auth, request-level conventions

**Goal:** Admin routes are gated by Cognito JWT (admin pool). Submission session routes still public but session-cookie based. Idempotency, ETag, rate-limit headers real.

**Phases (sketch):**

- **P1 — Wire `RequireAdmin` (from Platform's M2 work):** Every `/v1/admin/*` route gets the dependency. Assumes commons-pending or Platform-shipped `RequireAdmin`.
- **P2 — Wire `RequireSubmissionSession`:** A dependency that reads the `submission_session_id` HttpOnly cookie (or `X-Debug-Submission-Session-Id` falling back if dev), validates the session exists in DDB and is not revoked/expired, returns `SubmissionSessionContext`.
- **P3 — Org-membership and Workspace-membership checks:** AdminUser can only see Templates in Workspaces they have Memberships for. Filter all list/search queries by the AdminUser's accessible Workspace IDs.
- **P4 — Idempotency-Key:** Required on `POST /v1/admin/templates`, `POST /v1/admin/links`, `POST /v1/public/submission-sessions`, `POST /v1/public/submission-sessions/{id}/finalise`, and similar.
- **P5 — ETag + If-Match:** ETag on Template and Link GETs. `If-Match` required on PATCH/PUT of Template/Link. `If-Match` required on DELETE of Templates and Links (per §8.4 §6.6 high-value list).
- **P6 — Rate limiting:** Per-Org on `/v1/admin/*`, per-IP on `/v1/public/*`.
- **P7 — `Sunset` / `Deprecation` middleware:** Same as Platform M2 P9 — exists, emits nothing for `/v1/` yet.

**Decisions to grill:**
- How `RequireSubmissionSession` reads the cookie (FastAPI's cookie auth) — confirm cookie name (`insdi_submission_session_id` is conventional)
- Session expiry — TTL on the DDB item, how revocation works
- For `/v1/public/*` routes, the rate-limit bucket is per-IP (not per-Org since there's no Org yet at session-create time). Confirm bucket sizing

---

### M3 — Audit emission and observability

**Goal:** Audit events on Template/Link/Submission mutations. Tier C on reads. `X-Request-Id` everywhere.

**Phases (sketch):**

- **P1 — Wire `emit_with_tx` on writes:** template.create, template.update, template.delete, link.create, link.update, link.delete, submission.finalise. Tier A.
- **P2 — Standalone emission:** submission session start, field-changes (Tier B — sensitive content; HIPAA-flagged orgs only emit; otherwise Tier C). This is genuinely tricky — confirm the tier-decision logic during discuss.
- **P3 — Tier C reads:** template.list, link.list, submission.list, etc.
- **P4 — TransactWriteItems for field changes:** If P7 of M1 used two separate DDB writes, this is where it gets fixed to atomic (per §8.3). May involve commons-pending DDB helpers.
- **P5 — `X-Request-Id` end-to-end:** Same as Platform M3 P4.

---

### M4 — Production hardening

Same shape as platform-service M4: RDS Proxy, Secrets Manager, BaseSettings, SAM tuning, deploy to af-south-1 staging, OpenAPI export.

DDB tables also become production-shaped here: on-demand capacity, KMS encryption, TTL configuration.

---

### M5+ — Advanced Gather features

Suggested order:

- **EventBridge publication:** `template.created`, `template.published`, `submission.finalised`, etc., per §8.2 §3.7. Producer-side schema validation.
- **Template publish / versioning flow:** `POST /v1/admin/templates/{tpl_id}/publish` mints an immutable version; subsequent PATCHes target a draft version. Links can pin to a specific Template version. This is a substantial feature.
- **Link-scoped JWT integration:** Verified-identity submissions get a link-scoped JWT issued by Platform's `commons.platform_auth.issue_link_scoped_jwt`. Gather validates it on each request via `RequireSubmissionSession` extension.
- **File attachments:** Requires Resources to exist. Out of scope until Resources Lambda is built.
- **Allowlist-tag mechanism:** The `tag:vendor` allowlist resolution per the auth doc §7.1 — "submitters whose previous Submissions to this Org carry the vendor tag." Requires the Submission table to support tags.
- **Re-contact flow:** Per the auth doc §7.1 — every interaction is a Submission, even follow-up. Verify-flow integration creates new Links scoped to a single follow-up.
- **MCP surface:** Co-located with REST per §8.4 §9.3. Tools: `templates.create`, `templates.publish`, `submissions.search`, etc. Naming `noun.verb` matching REST.
- **Erasure participation:** Subscribe to `enduser.erasure-requested`, anonymise identity fields on Submissions, retain bodies per regulatory obligation, emit `enduser.erasure-completed` per the auth doc §7.4.
- **Tag-and-search-by-tag for submissions.**
- **Submission body validation hardening:** Schema validation that produces structured `errors` arrays per §8.4 §5.3 multi-error response shape.

---

## 4. Key §8.4 conventions Gather must follow

### URL surface (M1)

```
POST   /v1/admin/templates
GET    /v1/admin/templates
POST   /v1/admin/templates/search
GET    /v1/admin/templates/{tpl_id}
PATCH  /v1/admin/templates/{tpl_id}
DELETE /v1/admin/templates/{tpl_id}

POST   /v1/admin/links
GET    /v1/admin/links
POST   /v1/admin/links/search
GET    /v1/admin/links/{link_id}
PATCH  /v1/admin/links/{link_id}
DELETE /v1/admin/links/{link_id}

GET    /v1/admin/submissions
POST   /v1/admin/submissions/search
GET    /v1/admin/submissions/{sub_id}

POST   /v1/public/submission-sessions
POST   /v1/public/submission-sessions/{id}/field-changes
POST   /v1/public/submission-sessions/{id}/finalise
GET    /v1/public/submission-sessions/{id}              # session state read for the submitter

GET    /v1/home/submissions                             # EndUser sees their own
GET    /v1/home/submissions/{sub_id}                    # only if caller is the submitter
```

### URL surface (M5+)

```
POST   /v1/admin/templates/{tpl_id}/publish              # action — creates new version

# MCP tools (when MCP lands):
# templates.create, templates.get, templates.update, templates.publish
# links.create, links.get, links.update
# submissions.search, submissions.get
```

### Naming discipline

- `templates` not `gather-templates` — domain vocabulary per §8.4 §1.3
- `submission-sessions` kebab-case
- `field-changes` kebab-case
- ID prefixes: `tpl_`, `link_`, `sub_`, `subsess_` (or whatever convention — confirm during discuss)

### Computed-field discipline

Template, Link, and Submission responses **all** carry both raw declared and effective computed values where applicable:

```python
class TemplateResponse(BaseModel):
    id: UUID
    workspace_id: UUID
    name: str
    schema: dict  # field definitions
    # Raw declared
    auth_required_floor: AuthFloor | None
    submitter_allowlist: SubmitterAllowlist | None
    # Effective (computed by walking up the chain)
    effective_auth_floor: AuthFloor
    effective_allowlist: SubmitterAllowlist
    version: int
    created_at: datetime
    updated_at: datetime

class LinkResponse(BaseModel):
    id: UUID
    template_id: UUID
    short_id: str
    auth_required_floor: AuthFloor | None
    submitter_allowlist: SubmitterAllowlist | None
    effective_auth_floor: AuthFloor       # computed walking Link → Template → WS → Org
    effective_allowlist: SubmitterAllowlist
    # ...
```

This is the §8.2 / PJ's-memory pattern. Build it correctly from M1 P3 or it threads pain through every milestone.

---

## 5. The cross-service reads pattern (important)

Gather reads Platform-owned tables (Organisation, Workspace) directly via SQL on the shared `public` schema, per §8.2 §1.4. **Gather never writes them.**

In code:

```python
# Yes — direct SQL read
class TemplateService:
    def resolve_effective_policy(self, template_id):
        # Read Template (Gather-owned)
        template = self.session.get(Template, template_id)
        # Read Workspace (Platform-owned) via direct SQL — same DB connection
        workspace = self.session.get(Workspace, template.workspace_id)
        # Read Organisation (Platform-owned) via direct SQL
        org = self.session.get(Organisation, workspace.organisation_id)
        # ...walk the chain
```

This is permitted because it's in-process — same DB connection. There is no cross-Lambda call.

**Forbidden patterns:**
- HTTP call to `insdi-platform-service` to read a Workspace
- Direct Lambda-to-Lambda invocation of platform-service
- Writing to a Platform-owned table

The ORM definitions for these read-only entities live in `_commons_pending/models/` (or `insdi_commons.models` if commons has them) — ask the user.

---

## 6. M1 acceptance criteria

When M1 is done, a human can run this script end-to-end with curl:

```bash
# Setup (presumes Platform M1 has been run, OR Gather has created a Workspace stub locally)
WORKSPACE_ID=ws_01HXX...   # from Platform M1, or stubbed

# Create a Template
TPL=$(curl -X POST http://localhost:8000/v1/admin/templates \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d "{\"workspace_id\": \"$WORKSPACE_ID\", \"name\": \"Test Form\", \"schema\": {...}}" \
    | jq -r .id)

# Create a Link to it
LINK=$(curl -X POST http://localhost:8000/v1/admin/links \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d "{\"template_id\": \"$TPL\"}" \
    | jq -r .short_id)

# Public: start a submission session
SESSION=$(curl -X POST http://localhost:8000/v1/public/submission-sessions \
    -H 'Content-Type: application/json' \
    -d "{\"link_short_id\": \"$LINK\"}" \
    | jq -r .id)

# Public: write field changes
curl -X POST http://localhost:8000/v1/public/submission-sessions/$SESSION/field-changes \
    -H "X-Debug-Submission-Session-Id: $SESSION" \
    -H 'Content-Type: application/json' \
    -d '{"changes": [{"field_id": "name", "value": "Alice"}, {"field_id": "email", "value": "alice@example.com"}]}'

# Public: finalise
curl -X POST http://localhost:8000/v1/public/submission-sessions/$SESSION/finalise \
    -H "X-Debug-Submission-Session-Id: $SESSION"

# Admin: read it back
curl http://localhost:8000/v1/admin/submissions \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    | jq
```

Plus the §8.4 conventions are visible in the responses: cursor pagination, `_deprecation` absent (v1 not deprecated), error envelopes RFC 9457, computed effective fields on Template and Link responses, no `total` count anywhere.

---

## 7. Things to watch / known tensions

- **Policy-chain walk performance:** Walking Template → Workspace → Org on every Link response is one extra join. Fine at v1 scale. If list endpoints become slow, eager-load via SQLAlchemy. Don't over-engineer in M1.
- **DDB session state in M1:** M1 uses simplified writes (non-atomic) for field changes. **This is acceptable provided it's documented and fixed in M3.** Don't let it slip past.
- **Template versioning is deferred to M5+.** M1 stores a `version` integer that's always 1. The data model must support versioning when it lands, so the Template ORM should already have the column (versioning behaviour comes later, but the schema accommodates it from M1).
- **File attachments don't work until Resources exists.** The Template schema may declare `file` field types, but a submission with a file attachment can only be stubbed in M1. Decide during discuss whether to allow file fields in M1 (with a placeholder) or reject them.
- **Submission ↔ Verify / Calculate cross-service flow** is entirely M5+ work. Gather M1 finalises submissions in isolation; nothing happens downstream.
