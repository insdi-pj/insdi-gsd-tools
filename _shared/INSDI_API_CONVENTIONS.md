# insdi API Conventions (Distilled from §8.4)

**Status:** Quick-lookup reference; defers to §8.4 for any ambiguity
**Canonical source:** §8.4 API Surface — REST + MCP at `https://www.notion.so/36bc041b93b481aa85dcde3a1db13d74`

This document captures the §8.4 rules that come up most often during implementation. **Fetch §8.4 directly** for anything not covered here, or any case where this document's summary feels under-specified.

---

## URL structure

```
/v{N}/{namespace}/{resource}[/{id}[/{sub-resource}[/{sub-id}...]]]
```

- `v1` is the version prefix — **always present** on every route from M1 P1
- `namespace` is one of: `admin`, `home`, `public`, `auth`, `.well-known`
- `resource` is **plural-noun kebab-case** — `templates`, `submission-sessions`, `agent-grants`, `verified-domains`, `flow-runs`
- Resource names use **domain vocabulary, not service vocabulary**: it's `templates` (not `gather-templates`); `flows` (not `verify-flows`); `workbooks` (not `calculate-workbooks`)
- IDs are typed-prefix + UUIDv7: `tpl_01HXX...`, `sub_01HXX...`, `ws_01HXX...`, `org_01HXX...`, `au_01HXX...`, `eu_01HXX...`, `wb_01HXX...`, `flow_01HXX...`, `run_01HXX...`

## Namespaces

| Prefix | Auth dependency | Principal types |
|---|---|---|
| `/v1/admin/*` | `RequireAdmin` | AdminUsers, Agents |
| `/v1/home/*` | `RequireEndUser` | EndUsers |
| `/v1/public/*` | `RequireSubmissionSession` | Anonymous, verified-identity |
| `/v1/auth/*` | mixed / none | Anyone |
| `/.well-known/*` | none | Anyone |

In M1, the auth dependencies are placeholders that read `X-Debug-Principal-Id` instead of validating JWTs. Routes still go under the correct namespace from day one. MCP tools in M1 read the equivalent principal from `INSDI_DEBUG_PRINCIPAL_ID` env var set at server start.

A note on namespaces and MCP: MCP tools don't carry a namespace prefix in their name (a tool is `templates.create`, not `admin.templates.create`). In M1–M3 with the single `/mcp` endpoint per service, this is fine because the entire endpoint serves admin-equivalent tools. In M4+ the per-audience server split (`mcp.insdi.com/{admin,home,public}` per §8.4 §9.1) is what carries audience separation; tool names stay flat.

## Path conventions

- **Nested paths** when child has no meaningful identity outside parent and client is enumerating:
  - `GET /v1/admin/organisations/{org_id}/workspaces`
- **Flat paths with filters** when the child has its own ID:
  - `GET /v1/admin/submissions?workspace_id=...`
- **Actions are POST to noun-shaped sub-paths**, never verbs in the path:
  - Yes: `POST /v1/admin/templates/{tpl_id}/publish`
  - No: `POST /v1/admin/publish-template`

---

## Listing, filtering, sorting

### Cursor pagination

Every collection endpoint uses opaque cursor pagination:

```json
{
  "items": [...],
  "page": {
    "next_cursor": "eyJzb3J0Ijpb...",
    "prev_cursor": null,
    "limit": 50
  }
}
```

- Default `limit` = 50, max 200; exceeding max → `400 validation.limit_exceeded`
- Cursor is base64-encoded JSON; consumers treat as opaque
- Changing sort or filters between paged requests invalidates the cursor → `400 validation.invalid_cursor`
- **No `total` count.** No `/count` endpoint. Ever.

### GET filtering — operator suffixes

```
GET /v1/admin/submissions?workspace_id=ws_abc&status[in]=finalised,processing&created_at[gte]=2026-01-01
```

Operators: `[eq]` (default), `[ne]`, `[in]`, `[nin]`, `[gt]`, `[gte]`, `[lt]`, `[lte]`, `[contains]`, `[starts_with]`, `[is_null]`.

Combination semantics: AND across parameters, OR within `[in]`. No URL-level OR across different fields — that needs `POST /search`.

**Whitelisted-and-indexed discipline:** every filterable field is enumerated in the OpenAPI spec and indexed in Postgres. Filtering on a non-whitelisted field → `400 validation.field_not_filterable`.

### POST /search — JSON filter body

```json
POST /v1/admin/submissions/search
{
  "filter": {
    "and": [
      { "workspace_id": { "eq": "ws_abc" } },
      { "status": { "in": ["finalised", "processing"] } },
      { "or": [
        { "submitter_email": { "contains": "@acme.com" } },
        { "tags": { "contains": "high-priority" } }
      ]}
    ]
  },
  "sort": [{ "field": "created_at", "direction": "desc" }],
  "limit": 50,
  "cursor": null
}
```

- Max tree depth 4, max 20 leaf conditions → `400 validation.query_too_complex`
- `Cache-Control: no-store` on every `/search` response
- **Not every resource gets /search.** Small enumerations (Workspaces, Memberships, Organisations, AgentGrants) stay GET-only

### Sorting

- GET: sign-prefix syntax — `?sort=-created_at,name` (minus = descending)
- POST /search: `"sort": [{ "field": "created_at", "direction": "desc" }]`
- **Default sort across all user-facing endpoints:** `created_at desc, id desc` — this is part of the contract
- Sortable fields whitelisted and indexed, like filterable fields
- If client-supplied sort doesn't end in a unique field, server appends `id desc` for cursor correctness

---

## Response envelopes

### Single-resource success

Bare object. The body IS the resource.

```http
GET /v1/admin/templates/tpl_01HXX...

HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v3-d8a7c2..."
X-Request-Id: req_01HXX...

{
  "id": "tpl_01HXX...",
  "name": "Patient Intake",
  ...
}
```

No `{ data: ... }` wrapper, no `meta` block. Metadata goes in headers.

### Listing response

`{ items, page }` envelope as shown above.

### `_deprecation` envelope

The **only** middleware-injected top-level field on any response. Present only on responses from deprecated versions. Not relevant until vN+1 ships:

```json
{
  "id": "tpl_01HXX...",
  ...,
  "_deprecation": {
    "version": "v1",
    "sunset_date": "2027-11-15T00:00:00Z",
    "successor_version": "v2",
    "migration_guide_url": "https://docs.insdi.com/migrating-v1-v2"
  }
}
```

### Error envelope — RFC 9457

`Content-Type: application/problem+json`

```json
{
  "type": "https://docs.insdi.com/errors/policy.auth_floor_conflict",
  "title": "Auth-floor policy conflict",
  "status": 422,
  "detail": "Cannot set Workspace auth_required=anonymous when Org auth_required=authenticated",
  "instance": "/v1/admin/workspaces/ws_01HXX...",
  "code": "policy.auth_floor_conflict",
  "request_id": "req_01HXX..."
}
```

Multi-error responses extend with an `errors` array:

```json
{
  "type": "https://docs.insdi.com/errors/validation.multi",
  "title": "Validation failed",
  "status": 422,
  "detail": "Request body has 2 validation errors",
  "instance": "/v1/admin/templates",
  "code": "validation.multi",
  "request_id": "req_01HXX...",
  "errors": [
    { "code": "validation.field_required", "field": "name", "message": "name is required" },
    { "code": "validation.field_too_long", "field": "description", "message": "description exceeds 500 characters" }
  ]
}
```

### Error code taxonomy

Codes are snake_case with namespaced prefixes:

| Prefix | Use |
|---|---|
| `auth.*` | Authentication / authorisation failures |
| `validation.*` | Request shape / value validation |
| `policy.*` | Domain-policy violations |
| `conflict.*` | Concurrency / state conflicts |
| `not_found.*` | Resource not found |
| `rate_limit.*` | Throttling |
| `internal.*` | Server-side faults |

Adding new codes is additive. Renaming or removing is breaking.

### HTTP status mapping

| Status | When |
|---|---|
| 200 | Successful GET, PATCH, action returning a body |
| 201 | Successful POST creating a resource |
| 202 | Accepted for async processing |
| 204 | Successful DELETE; action with no body |
| 400 | Malformed request — bad JSON, missing required path/query params, invalid cursor |
| 401 | No / invalid / expired auth token |
| 403 | Authenticated but not authorised |
| 404 | Resource not found OR exists but caller has no visibility |
| 409 | Conflict — unique violation, invalid state transition |
| 410 | Permanently gone — sunset API versions |
| 412 | Precondition failed — `If-Match` mismatch |
| 422 | Request well-formed but semantically invalid |
| 428 | Precondition required — `If-Match` missing where required |
| 429 | Rate limited |
| 5xx | Server-side faults |

**Hard rule: 404 conflates "doesn't exist" with "you can't see it".** Never return 403 for "exists but not yours" — that leaks existence across Org boundaries.

---

## Idempotency

### `Idempotency-Key` required on creates and actions

Required on:
- POST resource creates (`POST /v1/admin/templates`, etc.)
- POST action endpoints with external side effects (`POST /v1/admin/templates/{id}/publish`, etc.)

Missing → `400 validation.idempotency_key_required`.

Optional on GET, DELETE, PATCH (PATCH uses `If-Match` instead).

### Server semantics

1. First request with this key for this Org → execute normally; persist `(org_id, idempotency_key, request_body_hash, response_status, response_body, expires_at)`
2. Retry with same key + same body hash → return cached response unchanged
3. Retry with same key + different body hash → `422 validation.idempotency_key_reused`

Retention: 24 hours. Table partitioned by `org_id`. Primary key `(org_id, idempotency_key)`.

**M1:** the header may be accepted but is not enforced; consider adding a `# TODO: enforce in M2` and accepting any value.
**M2:** full enforcement.

---

## Concurrency

### `ETag` + `If-Match`

Every single-resource GET returns `ETag` header (hash of `resource_version`, `updated_at`, `id`).

PATCH and PUT **require** `If-Match`:

| Condition | Response |
|---|---|
| `If-Match` missing | `428 validation.precondition_required` |
| `If-Match` provided, matches current | proceed; update returns new ETag |
| `If-Match` provided, does not match | `412 conflict.optimistic_lock` |

`If-Match: *` is accepted as "I don't care, just update" — audit-logged.

`If-Match` **required** on DELETE for: Templates, Workspaces, Flows, Workbooks, AgentGrants, FederatedIdpConfigs.

**M1:** ETags not generated; PATCH/PUT/DELETE accept request without `If-Match`.
**M2:** full enforcement.

---

## Rate limiting

### Buckets

| Bucket | Applies to |
|---|---|
| Per-Org | All `/v1/admin/*` and `/v1/home/*` once Org context resolves |
| Per-IP | All `/v1/public/*`, `/v1/auth/*`, `/.well-known/*` |
| Per-endpoint-class | OTP, sign-in, agent grants, federated IdP changes |
| Per-identity-attribute | OTP (per email, per phone) |

### Response headers

Every response includes:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1747500000
X-RateLimit-Bucket: org
```

`X-RateLimit-Bucket` values: `org`, `ip`, `endpoint:<class>`, `email`, `phone`. When multiple buckets apply, the headers reflect the **most-constrained** one.

### 429 response

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Bucket: org

{
  "type": "https://docs.insdi.com/errors/rate_limit.per_org",
  ...
  "code": "rate_limit.per_org",
  "retry_after_seconds": 12
}
```

**M1:** no rate limiting.
**M2:** full enforcement, DynamoDB-backed token buckets.

---

## Headers — quick reference (REST)

These apply to REST requests/responses. MCP has its own equivalents (per-call input parameters or session-level mechanisms) — see the MCP surface section below.

| Header | Direction | Required from | Purpose |
|---|---|---|---|
| `Authorization: Bearer ...` | request | M2 | Cognito JWT, link-scoped JWT, or agent token |
| `X-Debug-Principal-Id` | request | M1 only | Placeholder for principal in M1; removed in M2 |
| `Idempotency-Key` | request | M2 (on POST creates/actions) | Client-generated UUID |
| `If-Match: "<etag>"` | request | M2 (on PATCH/PUT, DELETE of high-value) | Optimistic concurrency |
| `Content-Type: application/json` | request | always | for JSON bodies |
| `X-Request-Id` | response | M3 | Request correlation; mirrors `request_id` in error envelopes |
| `ETag` | response | M2 (on single-resource GETs) | Concurrency control |
| `Sunset` | response | future (when v1 deprecated) | RFC 8594 |
| `Deprecation` | response | future | RFC 9745 |
| `X-RateLimit-*` | response | M2 | Rate limit state |
| `Retry-After` | response | M2 (on 429) | RFC 9110 |
| `Content-Type: application/problem+json` | response | M1 | All error responses |
| `Cache-Control: no-store` | response | M1 (on all `/search` responses) | Prevent stale search results |
| `Cache-Control: private, max-age=N` | response | M1 (default for authenticated reads) | Browser/CDN caching |

---

## Pydantic three-schema pattern

Per insdi convention (memory): every entity has three Pydantic schemas:

- `<Entity>CreateInput` — for POST request bodies; only client-providable fields
- `<Entity>UpdateInput` — for PATCH request bodies; partial, all fields optional
- `<Entity>Response` — for response bodies; includes computed/derived fields, server-generated IDs, timestamps

SQLAlchemy ORM rows convert to `<Entity>Response` via `model_validate(orm_row, from_attributes=True)`.

**Important nuance for entities with policy resolution** (Templates, Links, Workspaces, Organisations — applies to allowlist + auth-floor inheritance): the ORM stores **raw declared config** (`auth_required_floor`, `submitter_allowlist`); the response carries **computed effective fields** (`effective_auth_floor`, `effective_allowlist`) resolved by the service layer walking the Org → Workspace → Template → Link chain.

This pattern is established and consistent. The brief for each service spells out which entities need computed fields.

---

## Service Lambda anatomy

Every service Lambda repo has the same shape (established in M1 P1):

```
service-name/
├── README.md
├── pyproject.toml                  # FastAPI, SQLAlchemy v2, Pydantic v2, Alembic, boto3, fastapi-mcp or equivalent
├── template.yaml                   # SAM template
├── alembic.ini
├── migrations/
│   └── versions/
├── src/
│   └── <service_name>/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app, middleware, router and MCP server registration
│       ├── config.py               # Pydantic BaseSettings
│       ├── db.py                   # SQLAlchemy engine, session factory
│       ├── deps.py                 # FastAPI dependencies (RequireAdmin etc. — M2)
│       ├── errors.py               # Error envelope builder, RFC 9457 (used by both REST + MCP)
│       ├── pagination.py           # Cursor encode/decode, filter parser
│       ├── api/
│       │   └── v1/
│       │       ├── admin/          # /v1/admin/* REST routes
│       │       ├── home/           # /v1/home/* REST routes (only where applicable)
│       │       ├── public/         # /v1/public/* REST routes (only where applicable)
│       │       └── auth/           # /v1/auth/* REST routes (only where applicable)
│       ├── mcp/                    # MCP surface — co-located per §8.4 §9.3
│       │   ├── server.py           # MCP server setup, mounted at /mcp on the FastAPI app
│       │   ├── deps.py             # MCP-specific deps (UserContext resolution from session)
│       │   └── tools/              # One file per entity: organisations.py, workspaces.py, etc.
│       ├── models/                 # SQLAlchemy ORMs (or imports from insdi-commons)
│       ├── schemas/                # Pydantic models (or imports from insdi-commons) — used by REST AND MCP
│       ├── services/               # Use-case functions (business logic) — called by BOTH REST handlers AND MCP tools
│       └── _commons_pending/       # Temporary local impls of would-be-commons code
├── tests/
│   ├── services/                   # Use-case function tests (pure business logic)
│   ├── api/                        # REST handler tests
│   └── mcp/                        # MCP tool tests
└── .gitignore
```

The **use-case function** pattern (`services/`) is mandated by §8.4 §1.4 and is what makes MCP parity structurally enforceable. REST handlers in `api/` and MCP tools in `mcp/tools/` are both thin shape-adapters delegating to the same function in `services/`. The same Pydantic schemas in `schemas/` are used by both surfaces (REST as request/response body, MCP as `inputSchema`/`outputSchema`). Tests cover all three layers — the use-case function for business logic, the REST handler for HTTP-specific shape, the MCP tool for MCP-specific shape.

A new entity's CRUDLS phase therefore touches: `models/<entity>.py`, `schemas/<entity>.py`, `services/<entity>.py`, `api/v1/admin/<entity>.py`, `mcp/tools/<entity>.py`, `tests/services/<entity>.py`, `tests/api/<entity>.py`, `tests/mcp/<entity>.py`, plus a migration. That's deliberate — every layer is thin, but every layer is present.

---

## MCP surface

Per §8.4 §9 and `INSDI_GSD_PRINCIPLES.md` §3, MCP is co-located with REST in every service Lambda and is built alongside REST from M1 P1. This section captures the M1–M3 MCP-specific conventions; the M4+ split into per-audience servers, OAuth at session start, etc. is deferred per the principles doc.

### Transport

**Streamable HTTP only** (the current MCP transport per the 2025-03-26 MCP spec). Each service Lambda mounts an MCP server at `/mcp` on its FastAPI app. The same uvicorn process serves both REST (under `/v1/...`) and MCP (under `/mcp`).

No stdio. No SSE legacy.

### Tool naming — CRUDLS

Per project decision, MCP tools follow CRUDLS:

| MCP tool | Maps to REST |
|---|---|
| `<resource>.create` | `POST /v1/admin/<resource>` |
| `<resource>.read` | `GET /v1/admin/<resource>/{id}` |
| `<resource>.update` | `PATCH /v1/admin/<resource>/{id}` |
| `<resource>.delete` | `DELETE /v1/admin/<resource>/{id}` |
| `<resource>.list` | `GET /v1/admin/<resource>` |
| `<resource>.search` | `POST /v1/admin/<resource>/search` |

Note: `read` (not `get`) is the convention. Some MCP ecosystems use `get_x`; insdi uses `<resource>.read` to make CRUDLS read clearly.

Actions are tools named `<resource>.<verb>` where the verb is the action:

| MCP tool | Maps to REST |
|---|---|
| `templates.publish` | `POST /v1/admin/templates/{id}/publish` |
| `flow_runs.advance` | `POST /v1/admin/flow-runs/{id}/advance` |
| `tests.run` | `POST /v1/admin/tests/{id}/run` |

### URL → MCP namespace translation

The one consistent translation rule: **kebab-case URL resource → snake_case MCP namespace**.

| URL resource | MCP namespace |
|---|---|
| `templates` | `templates` |
| `flow-runs` | `flow_runs` |
| `agent-grants` | `agent_grants` |
| `verified-domains` | `verified_domains` |
| `workspace-memberships` | `workspace_memberships` |

This is because MCP tool names must be identifier-safe (no hyphens) per MCP convention.

### Tool input/output schemas

Tool input schemas are generated from the same Pydantic models used by REST. For example, the `templates.create` tool's `inputSchema` is generated from `TemplateCreateInput`:

```python
# schemas/templates.py
class TemplateCreateInput(BaseModel):
    workspace_id: UUID
    name: str
    description: str | None = None
    schema: dict
    auth_required_floor: AuthFloor | None = None
    submitter_allowlist: SubmitterAllowlist | None = None

# api/v1/admin/templates.py — REST
@router.post("/templates", status_code=201)
def create_template_route(
    input: TemplateCreateInput,
    ctx: UserContext = Depends(RequireAdmin),
) -> TemplateResponse:
    return services.templates.create_template(ctx, input)

# mcp/tools/templates.py — MCP
@mcp_tool(
    name="templates.create",
    description="Create a new Template in the specified Workspace. The Template's "
                "effective_auth_floor and effective_allowlist are computed by walking "
                "the policy chain (Org → Workspace → Template).",
)
def create_template_tool(
    input: TemplateCreateInput,
    ctx: UserContext,
) -> TemplateResponse:
    return services.templates.create_template(ctx, input)
```

Tool descriptions are part of the contract (§8.4 §9.7): they document insdi-specific guidance, security defaults, and any deviations from REST conventions. Tool descriptions ship from the codebase, not from a CMS — they're version-controlled.

### Listing tools — pagination

MCP `.list` and `.search` tools accept `cursor` and `limit` as input parameters, and return the same `{ items, page }` envelope as REST listing endpoints. The cursor is the same opaque base64 string used by REST — the same `pagination.encode_cursor`/`decode_cursor` helpers serve both surfaces.

```python
class ListTemplatesInput(BaseModel):
    cursor: str | None = None
    limit: int = 50
    workspace_id: UUID | None = None
    # plus operator-suffix-style filters when needed
```

### Error mapping

MCP returns errors as a content block with `isError: true`, containing the same RFC 9457 problem+json body REST uses:

```json
{
  "isError": true,
  "content": [
    {
      "type": "text",
      "text": "{\"type\":\"https://docs.insdi.com/errors/policy.auth_floor_conflict\",\"title\":\"Auth-floor policy conflict\",\"status\":422,\"detail\":\"...\",\"instance\":\"templates.create\",\"code\":\"policy.auth_floor_conflict\",\"request_id\":\"req_01HXX...\"}"
    }
  ]
}
```

The `instance` field for MCP errors uses the tool name (e.g. `templates.create`) instead of a URL path.

The `errors.problem_response` helper produces one shape; the surface adapters wrap it differently (REST returns it as `application/problem+json` HTTP body; MCP wraps it in the `isError: true` content block).

### Authentication (M1)

In M1, MCP sessions authenticate via an environment variable when the server starts:

```bash
INSDI_DEBUG_PRINCIPAL_ID=au_dev_1 uvicorn <service_name>.main:app --reload
```

Every MCP tool call uses this principal ID for `UserContext` construction. This is the MCP-side equivalent of the REST `X-Debug-Principal-Id` header. Removed in M2 when real OAuth-at-session-start lands.

### Local testing — MCP Inspector

Every service in M1 must be testable with MCP Inspector:

```bash
# Terminal 1: start the service with debug principal
INSDI_DEBUG_PRINCIPAL_ID=au_dev_1 uvicorn <service_name>.main:app --reload

# Terminal 2: launch MCP Inspector against the running service
npx @modelcontextprotocol/inspector http://localhost:8000/mcp
```

Inspector connects via streamable HTTP, enumerates tools, and provides an interactive UI for tool calls. M1 acceptance criteria for every service include "exercise every tool via MCP Inspector" alongside "exercise every REST route via curl."

### MCP-specific concerns deferred to M4+

These are §8.4 §9 features that don't apply per-tool — they're the equivalent of "the OpenAPI assembly pipeline" for REST. Not in scope M1–M3:

- Split into per-audience servers (`mcp.insdi.com/{admin,home,public}`) per §8.4 §9.1
- OAuth at MCP session start per §8.4 §8.1
- `/.well-known/mcp` discovery per §8.4 §10
- MCP session state in DynamoDB per §8.4 §9.9
- Build-time manifest assembly per §8.4 §11.7
- Prompts (curated workflow prompts) per §8.4 §9.8
- Resources (docs, policy summaries, catalogues) per §8.4 §9.6

In M1–M3, each service Lambda exposes its tools at `/mcp` on its own API; MCP Inspector connects directly. M4+ introduces the production multi-audience routing.

---

## Logging contract

Per §8.2 §8.5. Always-present fields on every log line: `ts`, `level`, `service`, `version`, `region`, `request_id`, `msg`. Authenticated requests also carry `ctx.{principal_type, actor_id, org_id, session_jti}`. Audit logs are NOT general logs — they go through `audit.emit_*`, not stdout.

**PII discipline:** never log submission body content, EndUser/AdminUser email or name (use IDs), tokens, API keys, file contents.

---

## What NOT to do — common temptations

### REST surface
- Don't add a `data: {...}` wrapper around single-resource responses
- Don't return `403` for "exists but not yours" — return 404
- Don't add a `total` count to listing responses (and don't add a `/count` endpoint)
- Don't use service vocabulary in URLs (`/v1/admin/gather/templates` ✗ → `/v1/admin/templates` ✓)
- Don't put verbs in paths (`/v1/admin/templates/publish/{id}` ✗ → `/v1/admin/templates/{id}/publish` ✓)
- Don't put metadata in response bodies that belongs in headers (`request_id` lives in `X-Request-Id`; the body only carries `request_id` in error envelopes)
- Don't conflate the audit log with structured logs
- Don't write to another service's owned tables (read via direct SQL OK per §8.2 §1.4; write via API or events)
- Don't add Cognito or auth code to M1 — it goes in M2
- Don't add audit emission to M2 — it goes in M3

### MCP surface
- Don't ship a REST route without its matching MCP tool in the same PR — parity violation
- Don't use `get` in tool names — use `read` per CRUDLS (`templates.read`, not `templates.get`)
- Don't use `verb_noun` MCP-community style — use `noun.verb` (`templates.create`, not `create_template`)
- Don't use hyphens in MCP tool names — use snake_case (`flow_runs.advance`, not `flow-runs.advance`); the REST path stays kebab-case
- Don't reimplement business logic inside an MCP tool — call the use-case function in `services/` that the REST handler calls
- Don't define a separate input schema for the MCP tool — reuse the Pydantic model from `schemas/` that the REST handler uses
- Don't return a different error shape from MCP than from REST — the same `errors.problem_response` helper produces both; MCP just wraps in `isError: true` content
- Don't ship a tool description that doesn't note insdi-specific deviations from MCP convention (the `noun.verb` order, `read` vs `get`, etc.) — agents reading tool docs need this context
- Don't add `prompts` or `resources` (MCP primitive types) in M1–M3 — they're M5+ content concerns
- Don't add OAuth-at-session-start to MCP in M1–M3 — M1 uses the `INSDI_DEBUG_PRINCIPAL_ID` env var, M2 will use real OAuth in M4+

