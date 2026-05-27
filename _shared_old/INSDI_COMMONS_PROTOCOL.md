# insdi-commons Protocol

**Status:** Authoritative for every insdi service GSD project
**See also:** `INSDI_GSD_PRINCIPLES.md` §4

`insdi-commons` is the shared Python (and TypeScript) package hosting cross-service code per §8.2 §1.1. Per the protocol:

> Not all of `insdi-commons` exists yet. When a service GSD project needs something that should logically live in `insdi-commons`, it asks the user whether to assume it exists or implement it locally.

This document is the concrete protocol.

---

## When to ask

Whenever the project needs a symbol (model, ORM table, dependency function, helper, constant, type) that satisfies any of:

- It is **owned by another service per §8.2 §2** but read by this one (e.g. Gather needs Platform's `Organisation` and `Workspace` ORMs to walk the policy chain)
- It is **explicitly listed in §8.2 §1.2** as living in commons (e.g. `RequireUserContext`, `audit.emit_with_tx`, the structured logger, link-scoped JWT issuance)
- It is **cross-cutting** by nature — used by more than one service if both existed (cursor pagination, error envelope builder, operator-suffix filter parser, `EntityRef`, `UserContext` and its subtypes, the event-envelope schema)

→ **Ask the user.**

When the symbol is unambiguously local to this service (a use-case function for a Template, an ORM table for `flow_steps`), don't ask — just implement.

---

## The question format

When asking, state explicitly:

1. **What the symbol is** (name, kind: model / function / class / constant)
2. **The shape needed** (signature, return type, side effects, key behaviour)
3. **Why this code should live in commons** (which point above applies)
4. **What the local fallback would be** if commons doesn't have it yet

Example:

> I need `insdi_commons.platform_auth.RequireAdmin`, a FastAPI dependency that validates a Cognito JWT against the admin pool, hydrates the AdminUser and its Memberships/WorkspaceMemberships from PG, and returns an `AdminUserContext`. This belongs in commons per §8.2 §1.2.
>
> Can I assume it exists in commons, or should I implement it locally in `_commons_pending/platform_auth.py`?

---

## Implementing locally — the convention

If the user says "implement locally":

### Location
All locally-implemented commons-pending code lives in:

```
src/<service_name>/_commons_pending/
```

Subdirectory structure mirrors the planned commons structure (e.g. `_commons_pending/audit/`, `_commons_pending/platform_auth/`, `_commons_pending/logging.py`).

### Marking
Every file inside `_commons_pending/` starts with this header comment:

```python
# TODO: Move to insdi-commons
# Target location in commons: insdi_commons.<module_path>
# Tracked in ROADMAP.md under "Migrate to insdi-commons"
#
# This is a temporary local implementation. The canonical home for this code
# is insdi-commons. Import sites elsewhere in this service should use:
#   from <service_name>._commons_pending.<module> import <symbol>
# When commons gains this code, all import sites swap to:
#   from insdi_commons.<module_path> import <symbol>
# and this file is deleted.
```

### Import discipline

Within the service, code that uses commons-pending symbols imports them with the `_commons_pending` path, not with `insdi_commons`. This makes the temporary nature visible in every use site and makes the eventual migration a mechanical find-and-replace.

```python
# Yes
from gather_service._commons_pending.audit import emit_with_tx

# No (this would silently break when the code moves to commons under a slightly
# different name or signature)
from insdi_commons.audit import emit_with_tx
```

### Roadmap entry

Add an entry to `ROADMAP.md` under a top-level section called **"Migrate to insdi-commons"** (create the section if it doesn't exist):

```markdown
## Migrate to insdi-commons

These symbols are currently implemented locally in `_commons_pending/` and should be moved to `insdi-commons` when that work is scheduled. Migration is mechanical: move the code, update imports, delete the local file.

| Local path | Target in commons | Description | Added in |
|---|---|---|---|
| `_commons_pending/platform_auth/require_admin.py` | `insdi_commons.platform_auth.RequireAdmin` | FastAPI dep: validate admin JWT, hydrate AdminUserContext | M2 P3 |
| `_commons_pending/audit/emit.py` | `insdi_commons.audit.emit_with_tx`, `emit_standalone` | Transactional and standalone audit emitters | M3 P1 |
```

The "Added in" column refers to the milestone/phase when the local impl was created. Useful for tracking accumulation.

---

## Migration phases

When `insdi-commons` later receives one of these symbols, the migration in this service is a dedicated phase, not bundled with feature work. The phase consists of:

1. Verify the commons signature matches the local impl's signature (if not, surface the divergence — don't silently adapt)
2. Update import sites (find/replace `<service>._commons_pending.<x>` → `insdi_commons.<x>`)
3. Delete the local file
4. Remove the ROADMAP entry
5. Run the full test suite — no test should change

If the commons signature differs from the local impl, a separate phase is needed first to either (a) bring the local impl in line with commons (and ship that as the migration), or (b) raise the difference to the user as a commons API conflict needing resolution.

---

## What does NOT go into `_commons_pending`

- Use-case functions specific to this service's domain (template create logic → `services/templates.py`, not `_commons_pending/`)
- ORMs for entities this service owns per §8.2 §2 (Templates ORM lives in `models/`, not `_commons_pending/`)
- Service-specific config, routes, schemas

The test: would a different service (Gather vs Verify vs Calculate) also need this exact code if it were doing similar work? If yes → commons-pending. If no → it's just this service's internal code.

---

## Frequently-needed commons symbols

Pre-emptive list of things that come up. For each, the typical question is: "is this in commons yet, or local?"

### Likely needed in M1

| Symbol | Purpose | §8.x reference |
|---|---|---|
| `commons.logging.get_logger(__name__)` | Structured logger configured with `service`, `version`, region | §8.2 §8.5 |
| `commons.pagination.encode_cursor`, `decode_cursor` | Cursor encode/decode + validation | §8.4 §4.1 |
| `commons.errors.problem_response(...)` | Build RFC 9457 problem+json error responses | §8.4 §5.3 |
| `commons.filters.parse_operator_suffixes(...)` | Parse `?status[in]=...` query params to SQLAlchemy filter clauses | §8.4 §4.2 |
| `commons.id.new_typed_id(prefix)` | Generate `tpl_01HXX...`-style typed UUIDv7 IDs | (insdi convention) |
| `commons.schemas.EntityRef` | `{ type, id }` used in audit event targets and event payloads | §8.2 §6 |
| ORMs for cross-service entities | Read-only access to Platform-owned tables from Gather/Verify/Calculate | §8.2 §2 |

### Likely needed in M2

| Symbol | Purpose | §8.x reference |
|---|---|---|
| `commons.deps.RequireAdmin`, `RequireEndUser`, `RequireAuth`, `RequireSubmissionSession`, `RequireAdminOrAgent` | FastAPI dependencies | §8.2 §6 |
| `commons.schemas.UserContext`, `AdminUserContext`, `EndUserContext`, `SubmissionSessionContext`, `AgentContext` | Discriminated UserContext family | §8.2 §6 |
| `commons.idempotency.idempotent(...)` decorator or middleware | Idempotency-key handling | §8.4 §6.1–6.3 |
| `commons.ratelimit.middleware` | DynamoDB-backed rate-limit middleware | §8.4 §7 |
| `commons.etag.compute(...)` | ETag generation | §8.4 §6.4 |
| `commons.middleware.request_id` | `X-Request-Id` generation/propagation | §8.4 §5.3 (`request_id` extension) |

### Likely needed in M3

| Symbol | Purpose | §8.x reference |
|---|---|---|
| `commons.audit.emit_with_tx(...)` | Transactional audit emission | §8.2 §5.1 |
| `commons.audit.emit_standalone(...)` | Standalone audit emission | §8.2 §5.2 |
| `commons.audit.emit_tier_c(...)` | Direct-DDB Tier C emission | §8.2 §5.3 |
| ORM/table definitions for `audit_outbox_transactional`, `audit_outbox_standalone` | Outbox tables | §8.2 §5 |

### Likely needed in M5+

| Symbol | Purpose | §8.x reference |
|---|---|---|
| `commons.platform_auth.issue_link_scoped_jwt(...)` | Sign link-scoped JWTs | §8.6 |
| `commons.federation.*` | Federated IdP helpers | §8.6 §2 |
| `commons.agents.intersect_permissions(...)` | Bounded-by-AdminUser invariant computation | §8.6 §8.4 |
| `commons.events.publish(...)` | EventBridge publisher with schema validation | §8.2 §3 |
| `commons.events.consume(...)` decorator | EventBridge consumer with idempotency check | §8.2 §3.5 |
| `commons.mcp.*` | MCP adapter helpers | §8.4 §9 |

---

## Summary

1. **Anything cross-service** → ask the user whether commons has it
2. **If commons doesn't have it** → implement in `_commons_pending/`, marked with the standard header, tracked in ROADMAP
3. **Imports use the `_commons_pending` path**, never `insdi_commons` directly until migration
4. **Migration is its own phase** — never bundled with feature work
