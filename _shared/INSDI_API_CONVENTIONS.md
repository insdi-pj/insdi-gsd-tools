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

In M1, the auth dependencies are placeholders that read `X-Debug-Principal-Id` instead of validating JWTs. Routes still go under the correct namespace from day one.

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

## Headers — quick reference

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
├── pyproject.toml                  # FastAPI, SQLAlchemy v2, Pydantic v2, Alembic, boto3
├── template.yaml                   # SAM template
├── alembic.ini
├── migrations/
│   └── versions/
├── src/
│   └── <service_name>/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app, middleware, router registration
│       ├── config.py               # Pydantic BaseSettings
│       ├── db.py                   # SQLAlchemy engine, session factory
│       ├── deps.py                 # FastAPI dependencies (RequireAdmin etc. — M2)
│       ├── errors.py               # Error envelope builder, RFC 9457
│       ├── pagination.py           # Cursor encode/decode, filter parser
│       ├── api/
│       │   └── v1/
│       │       ├── admin/          # /v1/admin/* routes
│       │       ├── home/           # /v1/home/* routes (only where applicable)
│       │       ├── public/         # /v1/public/* routes (only where applicable)
│       │       └── auth/           # /v1/auth/* routes (only where applicable)
│       ├── models/                 # SQLAlchemy ORMs (or imports from insdi-commons)
│       ├── schemas/                # Pydantic models (or imports from insdi-commons)
│       ├── services/               # Use-case functions (business logic; called by handlers)
│       └── _commons_pending/       # Temporary local impls of would-be-commons code
├── tests/
└── .gitignore
```

The **use-case function** pattern (services/) is mandated by §8.4 §1.4: route handlers are thin shape-adapters that delegate to use-case functions. This enables future MCP tool definitions to call the same use-case function the REST handler does, structurally enforcing "one state machine, two surfaces". Even before MCP arrives, this pattern is followed.

---

## Logging contract

Per §8.2 §8.5. Always-present fields on every log line: `ts`, `level`, `service`, `version`, `region`, `request_id`, `msg`. Authenticated requests also carry `ctx.{principal_type, actor_id, org_id, session_jti}`. Audit logs are NOT general logs — they go through `audit.emit_*`, not stdout.

**PII discipline:** never log submission body content, EndUser/AdminUser email or name (use IDs), tokens, API keys, file contents.

---

## What NOT to do — common temptations

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
