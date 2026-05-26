# GSD Project Brief — `insdi-platform-service`

**Lambda:** `insdi-platform-service`
**Domain:** Trust kernel — identity, auth, Organisations, Memberships, Workspaces, EndUser vault, link-scoped JWT issuance, erasure coordination
**Repo:** dedicated repo (separate GSD project)
**Status:** Brief for `/gsd-new-project`

**Read first:**
- `_shared/INSDI_GSD_PRINCIPLES.md` — the milestone/phase methodology
- `_shared/INSDI_API_CONVENTIONS.md` — §8.4 distilled
- `_shared/INSDI_COMMONS_PROTOCOL.md` — how to handle commons references

**Reference (fetch as needed during research/planning):**
- §8.1 System Overview — `https://www.notion.so/368c041b93b481a283b5d288ef550bdf`
- §8.2 Service Boundaries — `https://www.notion.so/368c041b93b48167a404fb0b9c0db2c5`
- §8.3 Data Architecture & Storage — `https://www.notion.so/35ec041b93b4814f9ecdf32f5ee92d37`
- §8.4 API Surface — REST + MCP — `https://www.notion.so/36bc041b93b481aa85dcde3a1db13d74`
- §8.6 Identity, Auth & Authorisation — `https://www.notion.so/35ac041b93b481169b79dd4734a63c05` (heavily relevant from M2 onward)
- Entities and Relationships — `https://www.notion.so/33dc041b93b480e98976f0280c5872dd`
- The Identity, Auth & Authorisation design (auth doc) — included in project files; this is the working design for M2+

---

## 1. Vision (for `PROJECT.md`)

`insdi-platform-service` is the foundation every other insdi service depends on. It owns the entities that define **who can do what** — Organisations, Memberships, Workspaces, AdminUsers, EndUsers — and the mechanisms through which those entities are authenticated and authorised.

Other services (`insdi-gather-service`, `insdi-verify-service`, `insdi-calculate-service`) call into Platform's data via direct SQL against the shared `public` schema (per §8.2 §1.4) and via the `insdi-commons` dependencies (per §8.2 §1.2). Platform's reliability is the most critical of all services — if Platform is down, no other service can construct a `UserContext` and all auth-requiring requests fail (§8.2 §9).

In v1 scope: REST API surface for managing the trust entities. Cognito integration for authentication. Federation (enterprise SSO). Agent OAuth flows. Link-scoped JWT issuance for submission paths. Domain verification. Erasure coordination.

Out of v1 scope (deferred): MCP surface, embeddable widget identity flows, programmatic API key management for Orgs, self-serve OAuth client registration.

---

## 2. Entities owned (from §8.2 §2)

Platform owns these entities. All other services read them via direct SQL (shared `public` schema); no one else writes them.

| Entity | Storage | Notes |
|---|---|---|
| `Organisation` | PG | Top-level tenant; holds `verified_domains`, `domain_restriction_enabled`, `auth_required_floor`, `submitter_allowlist`, `rate_limit_overrides` |
| `AdminUser` | PG | Org-employee identity; primary key is internal UUID; `cognito_sub` is unique secondary |
| `EndUser` | PG | home.insdi.com identity; same PK/cognito_sub pattern |
| `Membership` | PG | AdminUser ↔ Organisation, with role |
| `WorkspaceMembership` | PG | AdminUser ↔ Workspace, with role |
| `Workspace` | PG | Child of Org; holds optional `auth_required_floor`, `submitter_allowlist` |
| `VerifiedDomain` | PG | Org-claimed domain (DNS TXT / HTML / email-challenge verified) |
| `FederatedIdpConfig` | PG | Per-Org SSO IdP configuration (SAML/OIDC); maps to Cognito federated IdP |
| `AgentGrant` | PG | AdminUser's authorisation grant to an agent app client |
| Vault Entry | PG (PGVector) | EndUser-scoped pre-fill data |

The §8.2 watch-item flag on Vault is preserved: keep an eye on read load from Gather submission prefill.

---

## 3. Milestone roadmap

Follow the principles in `_shared/INSDI_GSD_PRINCIPLES.md`. The shape below is a starting point — refine during `/gsd-discuss-phase` for each milestone.

### M1 — Bare CRUD, no auth, conventions-correct

**Goal:** A FastAPI Lambda exposing the basic trust-entity CRUD under correct §8.4 URL structure, response shapes, error envelopes, and pagination. Routes accept `X-Debug-Principal-Id` header to inject an AdminUser ID; no JWT validation.

A human can:
- Create an Organisation
- Create a Workspace inside it
- Create AdminUsers and assign Memberships
- Create EndUsers
- List and search the above
- See proper RFC 9457 errors when things go wrong

**Phases (sketch — confirm in `/gsd-discuss-phase 1`):**

- **P1 — Scaffold:** Repo layout (per `INSDI_API_CONVENTIONS.md` "Service Lambda anatomy"), SAM template, FastAPI app skeleton, Postgres via Docker locally, Alembic baseline, structured logger, error envelope builder, cursor-pagination helper, operator-suffix filter parser. **Decide on the `_commons_pending/` convention here.**
- **P2 — Organisation CRUD:** Three-schema Pydantic (`OrganisationCreateInput`, `OrganisationUpdateInput`, `OrganisationResponse`); SQLAlchemy ORM; Alembic migration; routes `POST /v1/admin/organisations`, `GET /v1/admin/organisations`, `GET /v1/admin/organisations/{org_id}`, `PATCH /v1/admin/organisations/{org_id}`, `DELETE /v1/admin/organisations/{org_id}`. No `/search`. Tests for create, list (cursor pagination), update, delete, 404 conflation behaviour.
- **P3 — Workspace CRUD:** ORM, schemas, routes under `/v1/admin/organisations/{org_id}/workspaces` (nested per §8.4 §3.3 — Workspace has no identity outside Org for enumeration purposes; but `/v1/admin/workspaces/{ws_id}` exists for direct addressing). Foreign-key references to Organisation.
- **P4 — AdminUser + Membership:** AdminUser ORM/schemas, routes under `/v1/admin/admin-users`. Membership join table; routes under `/v1/admin/organisations/{org_id}/memberships`. The `X-Debug-Principal-Id` header now refers to AdminUser IDs.
- **P5 — WorkspaceMembership:** Routes for WS-level membership assignment under `/v1/admin/workspaces/{ws_id}/memberships`.
- **P6 — EndUser CRUD:** ORM, schemas, routes under `/v1/home/end-users/{id}` (note: home namespace because end users access this themselves) and `/v1/admin/end-users/{id}` (admin-side view; M1 still no real auth).
- **P7 — Search:** `POST /v1/admin/organisations/search`, `POST /v1/admin/admin-users/search`, `POST /v1/admin/end-users/search`. **Workspaces and Memberships are deliberately GET-only per §8.4 §4.3.**
- **P8 — Polish:** Documentation, README, end-to-end manual test script, OpenAPI schema sanity check (FastAPI auto-generates it; verify it conforms to §8.4 §1.3 naming).

**Decisions GSD should surface during `/gsd-discuss-phase 1`:**
- Three-schema Pydantic — confirm naming convention (`OrganisationCreateInput` vs `CreateOrganisation` vs `OrganisationCreate`); see PJ's memory for prior decisions
- `_commons_pending/` convention details (sub-structure, naming, header comment exact wording)
- Whether the Organisation has a `slug` field, what slug rules look like (DNS-safe? case? length?)
- Whether Workspace's parent is by URL nesting (`/organisations/{org_id}/workspaces`) only, or also by request body (`POST /workspaces` with `workspace.organisation_id`)
- Cursor encoding details — what fields encode in the cursor when sort is `created_at desc, id desc`; how to encode the case where a custom sort is applied

**Out of scope for M1 (deferred):**
- Any JWT validation
- Cognito user pools (no pools exist yet — there's nothing to validate against)
- Any audit emission
- ETags, If-Match, Idempotency-Key enforcement (header may be accepted, ignored, marked in OpenAPI as M2)
- Rate limiting
- RDS Proxy (M4); use direct Postgres locally
- Domain verification flows (M5)
- Federation, SSO routing (M5)
- Agent grants (M5)
- Link-scoped JWT signing (M5)
- Erasure coordination (M5+)
- Vault entries / PGVector (M5+)
- VerifiedDomain table — may exist but no verification logic (M5)
- FederatedIdpConfig table — may exist but no logic (M5)
- AgentGrant table — may exist but no logic (M5)

---

### M2 — Cognito JWT validation, real auth, request-level conventions

**Goal:** The M1 API now requires real authentication. `X-Debug-Principal-Id` is removed. Idempotency, ETag, rate-limit headers are real.

**Phases (sketch):**

- **P1 — Cognito user pools provisioned:** Either via stub local pool (cognito-local or moto) or against a real dev-environment Cognito pool. Two pools: `insdi-admin-pool`, `insdi-enduser-pool`, per §8.6 §1.3. Decide which approach during `/gsd-discuss-phase 2`.
- **P2 — JWKS caching, JWT validation:** Implementation of `RequireAuth` (token-only, no DB). Establishes the auth-validation infrastructure. **Likely lives in `_commons_pending/platform_auth/`** — ask the user.
- **P3 — `UserContext` hydration:** `RequireAdmin` validates the admin-pool token, hydrates AdminUser + Memberships + WorkspaceMemberships, returns `AdminUserContext`. `RequireEndUser` for the end-user pool. Discriminated by `iss` claim. Likely commons-pending.
- **P4 — Wire dependencies on existing routes:** Every existing M1 route gets the appropriate `Require*` dependency. The Org-membership check (caller can see Org X only if they have a Membership) is implemented here.
- **P5 — Cognito post-confirmation trigger:** When a Cognito user signs up, a corresponding AdminUser/EndUser record is mirrored into PG (per §8.6 §1.5). Implementation can be a stub Lambda or a polled-event handler — decide during discuss.
- **P6 — Idempotency-Key enforcement:** `Idempotency-Key` required on POST creates (Organisation, Workspace, AdminUser invitations, Membership creates). `idempotency_keys` table; 24h retention; per-Org partitioning per §8.4 §6.2.
- **P7 — ETag + If-Match:** ETag on single-resource GETs; `If-Match` required on PATCH/PUT and DELETE of high-value resources (Workspace, FederatedIdpConfig once introduced — for now Workspace and any other patchable resource).
- **P8 — Rate limiting:** Per-Org and per-IP buckets; DynamoDB-backed token buckets; `X-RateLimit-*` headers per §8.4 §7.5. Override mechanism reads from `Organisation.rate_limit_overrides` JSONB column.
- **P9 — `Sunset` / `Deprecation` middleware:** No deprecated versions yet, but the middleware exists; emits no header for `/v1/` (since v1 isn't deprecated yet).

**Decisions to grill in `/gsd-discuss-phase 2`:**
- Real Cognito vs stubbed local pool for dev — affects test approach
- Where the Cognito post-confirmation logic lives (separate Lambda? FastAPI endpoint that Cognito hits? polled?)
- Idempotency table substrate — §8.4 §15 flags an open question between Postgres and DynamoDB; §8.4 §6.2 currently specifies Postgres. PJ to confirm
- Rate-limit middleware: position in the FastAPI middleware stack; how it resolves the Org for `/v1/admin/*` (after auth?) and IP for `/v1/public/*` (before auth?)

**Out of scope for M2:**
- Federation / SSO (still M5)
- Agent grants (still M5)
- Domain verification (still M5)
- Link-scoped JWTs (still M5)
- Audit emission (M3)

---

### M3 — Audit emission and observability

**Goal:** Every write emits an audit event per §8.2 §5. `X-Request-Id` propagates end-to-end.

**Phases (sketch):**

- **P1 — Outbox tables + emitter helpers:** `audit_outbox_transactional`, `audit_outbox_standalone` PG tables; `commons.audit.emit_with_tx`, `emit_standalone` helpers. Likely commons-pending.
- **P2 — Wire emit_with_tx on existing write routes:** Every POST/PATCH/DELETE that mutates an entity now emits a Tier A or B audit event in the same transaction.
- **P3 — Tier C emission on reads:** `audit.emit_tier_c` for `*.list`, `*.read`, search reads. Direct DDB write, no PG outbox involvement.
- **P4 — `X-Request-Id` middleware:** Generate or accept a request ID on every request; propagate to logs, audit events, error envelope `request_id`. Persists from M1 if it was set up; otherwise it's wired here.
- **P5 — Structured exception logging:** Per §8.2 §8.5; unhandled exception handler in FastAPI emits structured JSON error log.

**Important gap (call out in `ROADMAP.md`):** `insdi-audit-service` (the outbox drainer Lambda) is NOT in the four-service scope. M3 writes audit events to the PG outbox but no drainer moves them to DynamoDB. This is acceptable for now per §8.2 §9 — the failure mode is "drainer lag → drainer catches up on recovery," which here means "lag until the audit-service is built." Until then, the audit table grows in PG and can be inspected directly.

**Decisions to grill in `/gsd-discuss-phase 3`:**
- The audit-event class list for Platform's owned entities (org.created, org.updated, workspace.created, etc.) — confirm the verb vocabulary
- Tier assignment for each event class (A / B / C)
- Audit payload schema — what goes in the `payload` field per event class

---

### M4 — Production hardening

**Goal:** RDS Proxy, Secrets Manager, `af-south-1` deployment to staging, OpenAPI assembly readiness.

**Phases (sketch):**

- **P1 — RDS Proxy wiring:** Switch from direct Aurora connection to RDS Proxy with IAM auth per `insdi-platform-service`'s memory; connection-pool tuning.
- **P2 — Secrets Manager:** DB creds, JWT signing keys, OAuth client secrets via Secrets Manager. Cold-start fetch + cache.
- **P3 — Pydantic BaseSettings + env config:** Final config shape per §8.2 §8.4.
- **P4 — SAM template tuned:** Cold-start performance, provisioned concurrency considerations.
- **P5 — Deploy to af-south-1 staging:** Validate the deployment pipeline.
- **P6 — OpenAPI export:** `/openapi.json` ready for the deploy-time assembler per §8.4 §11.

---

### M5+ — Advanced platform features (service-specific roadmap)

These layer in over multiple milestones. Order is suggested; refine when actually planning.

- **VerifiedDomain + domain verification flows** — DNS TXT / HTML meta / email-challenge per §8.6 §5.1; per-Org `domain_restriction_enabled` toggle and grandfathering UX per §8.6 §6.1–6.2
- **Federation (FederatedIdpConfig):** Self-serve SSO config via `AdminCreateIdentityProvider`; domain-based routing on login per §8.6 §2
- **Custom login page:** No Cognito hosted UI; domain-routed login per §8.6 §2.4
- **AgentGrant + OAuth Authorization Code with PKCE:** Endpoints `/v1/auth/oauth/{authorize, consent, token, revoke}` per §8.4 §8.2; bounded-by-AdminUser invariant per §8.6 §8.4 / §8.4 §8.4
- **Link-scoped JWT signing:** `commons.platform_auth.issue_link_scoped_jwt` (signing key in Platform-owned KMS); JWKS publication at `/.well-known/jwks.json` per §8.4 §10
- **Vault + PGVector:** EndUser pre-fill data
- **Erasure coordination:** Cross-service erasure event publishing and collection per §8.2 §3.7 (`enduser.erasure-requested` / `enduser.erasure-completed`)
- **MCP surface:** Co-located with REST per §8.4 §9.3; admin-pool and end-user-pool MCP servers; MCP session state in DynamoDB per §8.4 §9.9
- **Per-Org KMS keys:** For envelope encryption of sensitive fields; provisioned via `insdi-infra` CDK

---

## 4. Key §8.4 conventions Platform must follow

### URL surface (M1)

```
POST   /v1/admin/organisations
GET    /v1/admin/organisations
POST   /v1/admin/organisations/search
GET    /v1/admin/organisations/{org_id}
PATCH  /v1/admin/organisations/{org_id}
DELETE /v1/admin/organisations/{org_id}

GET    /v1/admin/organisations/{org_id}/workspaces
POST   /v1/admin/workspaces                       # workspace.organisation_id in body
GET    /v1/admin/workspaces/{ws_id}
PATCH  /v1/admin/workspaces/{ws_id}
DELETE /v1/admin/workspaces/{ws_id}

POST   /v1/admin/admin-users
GET    /v1/admin/admin-users
POST   /v1/admin/admin-users/search
GET    /v1/admin/admin-users/{au_id}
PATCH  /v1/admin/admin-users/{au_id}
DELETE /v1/admin/admin-users/{au_id}

GET    /v1/admin/organisations/{org_id}/memberships
POST   /v1/admin/memberships                      # membership.{admin_user_id, organisation_id, role} in body
DELETE /v1/admin/memberships/{membership_id}

GET    /v1/admin/workspaces/{ws_id}/memberships
POST   /v1/admin/workspace-memberships
DELETE /v1/admin/workspace-memberships/{ws_membership_id}

POST   /v1/admin/end-users                        # admin creates an end-user (rare)
GET    /v1/admin/end-users
POST   /v1/admin/end-users/search
GET    /v1/admin/end-users/{eu_id}

POST   /v1/home/end-users                         # end-user self-create (signup mirror)
GET    /v1/home/end-users/me
PATCH  /v1/home/end-users/me
```

### URL surface (M2+)

Wires real `RequireAdmin` and `RequireEndUser` to the above; introduces:

```
POST   /v1/auth/admin/sign-in                     # M2 — Cognito-fronted
POST   /v1/auth/enduser/sign-in                   # M2 — Cognito-fronted
GET    /v1/auth/admin/sso/callback                # M5 — federation
POST   /v1/auth/otp/request                       # M5 — verified-identity submission
POST   /v1/auth/otp/verify                        # M5
GET    /v1/auth/oauth/authorize                   # M5 — agent OAuth
GET    /v1/auth/oauth/consent                     # M5
POST   /v1/auth/oauth/consent                     # M5
POST   /v1/auth/oauth/token                       # M5
POST   /v1/auth/oauth/revoke                      # M5

POST   /v1/admin/agent-grants                     # M5
GET    /v1/admin/agent-grants
DELETE /v1/admin/agent-grants/{grant_id}          # If-Match required

POST   /v1/admin/federated-idps                   # M5
GET    /v1/admin/federated-idps
PATCH  /v1/admin/federated-idps/{idp_id}          # If-Match required
DELETE /v1/admin/federated-idps/{idp_id}          # If-Match required

POST   /v1/admin/organisations/{org_id}/verified-domains      # M5
POST   /v1/admin/verified-domains/{domain_id}/verify           # M5 (action)

GET    /.well-known/openapi.json                  # M4
GET    /v1/openapi.json                           # M4
GET    /.well-known/jwks.json                     # M5
GET    /.well-known/oauth-authorization-server    # M5
GET    /.well-known/mcp                           # MCP milestone (later)
GET    /.well-known/health                        # M1 or M2
```

### Resource-name discipline

`agent-grants` not `agentGrants` or `agent_grants`. `verified-domains`, `federated-idps`, `workspace-memberships`. Single-word resources: `organisations`, `workspaces`, `memberships`.

### Computed fields on the Organisation response

Per PJ's memory: ORM stores **raw declared config**, response carries **computed effective fields**. For Organisation in M1, there is no parent — so `auth_required_floor` declared = `effective_auth_floor` trivially. But the schema separation must exist from M1, because Workspace responses will use the same pattern non-trivially in M5+. Build the pattern correctly in M1 even though the computation is identity.

```python
class OrganisationResponse(BaseModel):
    id: UUID
    name: str
    slug: str | None
    # Raw declared values
    auth_required_floor: AuthFloor | None
    submitter_allowlist: SubmitterAllowlist | None
    # Computed values (in M1, equal to declared; in M5+, may differ)
    effective_auth_floor: AuthFloor
    effective_allowlist: SubmitterAllowlist
    # ...
```

For Workspace, the computation walks `Workspace → Organisation`. For Templates and Links (owned by Gather), the computation walks the full chain. The discipline is the same everywhere.

---

## 5. M1 acceptance criteria (the runnable bar)

When M1 is "done," a human with curl and a fresh local Postgres can:

1. `POST /v1/admin/organisations` with `X-Debug-Principal-Id: au_dev_1` and a valid body → returns 201 + Organisation response
2. `GET /v1/admin/organisations/{id}` → returns 200 + the same Organisation
3. `GET /v1/admin/organisations` → returns 200 + `{ items, page }` envelope with cursor pagination
4. `GET /v1/admin/organisations?name[contains]=foo&created_at[gte]=2026-01-01` → returns 200 with filter applied
5. `POST /v1/admin/organisations/search` with a complex `filter` body → returns 200; complexity > limit → `400 validation.query_too_complex`
6. `GET /v1/admin/organisations/does_not_exist` → returns `404 not_found.organisation` with proper RFC 9457 envelope
7. `POST /v1/admin/organisations` with bad JSON → returns `400` with proper envelope
8. `POST /v1/admin/organisations` missing a required field → returns `422 validation.multi` with `errors` array
9. All routes show correct `Content-Type` (`application/json` for success, `application/problem+json` for errors)
10. Workspaces, AdminUsers, EndUsers, Memberships all work the same way
11. The OpenAPI doc at `/openapi.json` (FastAPI's auto-generated one) shows all routes correctly namespaced and named

No JWT validation. No audit. No idempotency key required. No rate limiting. No ETag.

---

## 6. Things to watch / known tensions

- **Vault is in Platform but read by Gather** (§8.2 §2 watch item). Until Gather is built, this doesn't matter. When Gather adds submission prefill, watch the load.
- **The auth doc** (in project files) is the working design for §8.6. M2 onward references it heavily. Some details (link-scoped JWT mechanics, OTP, step-up auth) are flagged as open questions in the auth doc itself — when implementing, surface those decisions during discuss.
- **The "no separate auth service" decision (auth doc §1.1)** means Platform houses the auth module. In M1, the auth module doesn't exist yet. In M2 it gets created. In M5+ the federation/agent/link-scoped JWT logic accumulates inside it.
- **Resources and Audit Lambdas are out of scope per PJ.** Where Platform would normally publish events that Resources or Audit consume, those events are still published (M5+ for events generally), but no consumer exists. This is documented and accepted.
