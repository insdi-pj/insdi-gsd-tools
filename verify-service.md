# GSD Project Brief — `insdi-verify-service`

**Lambda:** `insdi-verify-service`
**Domain:** Workflow orchestration — Flows, FlowSteps, FlowRuns
**Repo:** dedicated repo (separate GSD project)
**Status:** Brief for `/gsd-new-project`

**Read first:**
- `_shared/INSDI_GSD_PRINCIPLES.md` — the milestone/phase methodology
- `_shared/INSDI_API_CONVENTIONS.md` — §8.4 distilled
- `_shared/INSDI_COMMONS_PROTOCOL.md` — how to handle commons references

**Reference:**
- §8.2 Service Boundaries — `https://www.notion.so/368c041b93b48167a404fb0b9c0db2c5`
- §8.4 API Surface — REST + MCP — `https://www.notion.so/36bc041b93b481aa85dcde3a1db13d74`
- Entities and Relationships — `https://www.notion.so/33dc041b93b480e98976f0280c5872dd`

---

## 1. Vision (for `PROJECT.md`)

`insdi-verify-service` is the workflow engine. A **Flow** is a versioned definition of a multi-step process applied to Submissions — for example: *"on KYC submission, fetch credit bureau report, route to manual review if score below threshold, notify the submitter of outcome."* A **FlowStep** is one step within that definition. A **FlowRun** is one execution of a Flow against a specific Submission, with its own state machine and outcome.

Verify sits downstream of Gather: in production, the `submission.finalised` event from Gather kicks off any bound FlowRuns. The Flow then orchestrates calls to external systems, conditional logic, possibly Calculate Workbooks (computation), possibly re-issued Gather Links (for follow-up data), and produces a final verdict.

In v1 scope: REST API surface for designing and managing Flows. FlowRun lifecycle CRUD. Manual step advancement (an AdminUser advancing a FlowRun). The event-driven start-from-submission path is M5+.

---

## 2. Entities owned (from §8.2 §2)

| Entity | Storage | Notes |
|---|---|---|
| `Flow` (versioned) | PG | The workflow definition; like Templates, immutable past versions |
| `FlowStep` | PG | Individual step definition within a Flow |
| `FlowRun` | PG | An execution instance bound to a specific Submission |

**Cross-service reads (direct SQL):**

- `Organisation`, `Workspace`, `Membership`, `WorkspaceMembership` (Platform-owned) — same pattern as Gather
- `Submission`, `Template` (Gather-owned) — a FlowRun's input
- `CalculateRun` (Calculate-owned) — a FlowStep may bind a Calculate run

**Cross-service writes — forbidden.** When a FlowStep needs to mint a new Gather Link (for follow-up data), the design (per §8.2 §3.7) is to publish a `verify.link-requested` event, which Gather consumes and acts on. **This is M5+ work** — M1 has no cross-service writes.

---

## 3. Milestone roadmap

### M1 — Bare CRUDLS, no auth, conventions-correct, MCP parity

**Goal:** A FastAPI Lambda where a human can define a Flow with several Steps, manually create a FlowRun against a Submission, and advance the run through its steps — on **both** REST and MCP surfaces. All under §8.4 conventions. No authentication.

**Phases (sketch — confirm in `/gsd-discuss-phase 1`):**

Each entity-phase delivers REST routes AND matching MCP tools in the same PR per principles doc §3.

- **P1 — Scaffold:** Same template as Platform and Gather P1, including `mcp/` directory and MCP server mounted at `/mcp` via streamable HTTP. Follow the conventions and library choices already locked by Platform M1 P1 — copy, don't reinvent. Smoke-test: MCP Inspector connects and lists 0 tools.

- **P2 — Cross-service ORMs (read-only):** Define Gather's Submission ORM and Platform's Organisation/Workspace ORMs for reading. `_commons_pending/models/` — ask the user. No tools.

- **P3 — Flow CRUDLS:**
  - Three-schema Pydantic. Flow ORM with `flow_id`, `workspace_id`, `name`, `description`, `version` (always 1 in M1), `definition` (JSONB — step graph), `created_at`, `updated_at`.
  - Use-case functions in `services/flows.py`: `create_flow`, `read_flow`, `update_flow`, `delete_flow`, `list_flows`, `search_flows`.
  - REST routes: `POST /v1/admin/flows`, `GET`, `GET /{id}`, `PATCH /{id}`, `DELETE /{id}`, `POST /search`.
  - MCP tools: `flows.{create, read, update, delete, list, search}`.

- **P4 — FlowStep CRUD:** Two design options to grill during discuss:
  - **Option A:** FlowSteps are embedded in the Flow's `definition` JSONB — no separate routes or tools; you edit a Flow's steps by PATCHing the Flow / calling `flows.update`
  - **Option B:** FlowSteps are first-class entities with their own routes and MCP tools (`flow_steps.{create, read, update, delete, list}`)
  
  Option A is simpler — no FlowStep tool surface to maintain. Option B has finer-grained surface. **Recommend A for M1**, reconsider B in M5+ if needed.

- **P5 — FlowRun creation:** 
  - REST: `POST /v1/admin/flow-runs` accepts `{ flow_id, submission_id }`, creates a FlowRun row with `status=running`, `current_step_id=<first_step>`, `started_at`.
  - MCP: `flow_runs.create` tool with same input/output. **In M1 no actual step execution happens** — the FlowRun just exists in a state machine.

- **P6 — FlowRun advancement:**
  - REST: `POST /v1/admin/flow-runs/{run_id}/advance` — manually advance to the next step. Updates `current_step_id`. Sets `status=completed` if at the end.
  - MCP: `flow_runs.advance` tool.
  - Tool description: "Manually advance the FlowRun to the next step. In M1 this is the only execution mechanism; M5+ adds automated step execution via background workers and event triggers."

- **P7 — FlowRun listing/search/read:**
  - REST: `GET /v1/admin/flow-runs`, `POST /v1/admin/flow-runs/search`, `GET /v1/admin/flow-runs/{run_id}`.
  - MCP: `flow_runs.{list, search, read}`. (Note: `read` not `get`.)

- **P8 — Polish:** Manual test script for REST (curl) and MCP (Inspector session). OpenAPI + `tools/list` sanity checks.

**Decisions to grill in `/gsd-discuss-phase 1`:**
- **Confirm the Flow `definition` field uses the typed `insdi-commons` model, not `dict`.** Per principles doc §4, the FlowDefinition shape (steps, transitions, step-type config) comes from `commons.schemas.FlowDefinition` — so MCP's `inputSchema` for `flows.create` carries the full structure (allowed step types, type-specific config via discriminated union) and AI clients can produce valid input on the first try. If commons' current `FlowDefinition` doesn't cover what insdi v2 needs, schedule a commons update — don't shadow locally with `dict`.
- Embedded vs first-class FlowSteps (Option A vs B above)
- Which step types `FlowDefinition` (in commons) supports today vs which are explicitly M5+. Given no real execution in M1, a `manual` step type may be the only one needed; `integration_call`, `branch`, `calculate_run`, `gather_followup` are M5+ regardless
- FlowRun `status` enum — `running`, `completed`, `failed`, `cancelled`? Add `awaiting_input` and `paused`?
- Whether `current_step_id` is a denormalised field or computed from FlowRun's event history (M1 should denormalise — simpler)
- ID prefixes: `flow_`, `flowstep_` (if first-class), `run_`
- MCP tool naming for `flow_runs.advance` — confirm; or `flow_runs.next_step`?

**Out of scope for M1:**
- JWT validation, auth (M2)
- Audit (M3)
- Idempotency, ETag, rate-limit (M2)
- Actual step execution (M5+) — M1 only advances the state machine manually
- EventBridge consumption (`submission.finalised` trigger) — M5+
- EventBridge publication (`flow.completed`, etc.) — M5+
- Cross-service writes via events (`verify.link-requested`) — M5+
- Integration calls to external systems — M5+
- Bound CalculateRun integration — M5+
- Conditional branching in Flow definitions — M5+
- MCP-specific production concerns (per-audience server split, OAuth, discovery) — M4

---

### M2 — Cognito JWT validation, real auth

**Goal:** Admin routes gated. Standard request-level conventions enforced.

**Phases (sketch):**

- **P1 — Wire `RequireAdmin` on every existing route.**
- **P2 — Workspace-membership filtering:** AdminUser sees Flows in Workspaces they have access to.
- **P3 — Idempotency-Key on creates and advance actions.**
- **P4 — ETag + If-Match:** ETag on Flow GETs. `If-Match` required on PATCH/DELETE of Flow (§8.4 §6.6 high-value).
- **P5 — Rate limiting.**
- **P6 — `Sunset` / `Deprecation` middleware.**

---

### M3 — Audit emission

- **P1 — `emit_with_tx` on Flow/FlowRun mutations.**
- **P2 — Tier C on reads.**
- **P3 — Particular attention to FlowRun lifecycle events** — `flowrun.started`, `flowrun.advanced`, `flowrun.completed`, etc., as Tier A.
- **P4 — `X-Request-Id`.**

---

### M4 — Production hardening

Same shape as Platform and Gather M4.

---

### M5+ — Advanced Verify features

- **EventBridge consumption:** Subscribe to `submission.finalised`; if any Flow is bound to the submission's Template, automatically start a FlowRun. **Idempotency on `event_id` is critical** (§8.2 §3.5).
- **EventBridge publication:** `flow.completed`, `flow.failed`, `verify.link-requested` (for follow-up Gather Links).
- **Actual step execution:** A worker pattern — Lambda invocation per advance, or step-functions integration. Major design decision; defer until requirements are crisp.
- **Flow versioning / publish flow:** Like Templates, `POST /v1/admin/flows/{flow_id}/publish` mints an immutable version.
- **Step types — integration calls:** Outbound HTTP to external systems with retry, timeout, error mapping.
- **Step types — Calculate binding:** A step that triggers a Calculate Workbook and waits for its result.
- **Step types — Gather follow-up:** A step that publishes `verify.link-requested`; Gather mints a Link and emails the original submitter. The follow-up Submission flows back as input to a subsequent step.
- **Conditional branching:** Step transitions depend on data values, comparison results, calculate output, etc.
- **Manual-review steps:** A step that pauses the run until an AdminUser approves/rejects.
- **`flows.draft_from_description` (M6+, post-stability):** A natural-language → FlowDefinition drafting tool. Admin says *"verify ID with credit-bureau check; manual review if score below threshold; notify submitter of outcome"*; the tool returns a proposed `FlowCreateInput` with rationale and alternatives, which the admin reviews before calling `flows.create`. Backed by insdi-controlled LLM call. **Requires:** typed `FlowDefinition` proven in production, a corpus of successful Flows for few-shot examples, evaluation harness. See `INSDI_GSD_PRINCIPLES.md` §4 for the full drafting-tool progression rationale.
- **MCP enhancements (M5+ adds new tools as features land):** Each M5+ feature ships its REST routes and matching MCP tools together. `flows.publish` (when versioning lands), event-driven `flow_runs.create` would be triggered externally (no MCP tool — it's an event handler), etc.

---

## 4. Key §8.4 conventions

### URL surface (M1)

```
POST   /v1/admin/flows
GET    /v1/admin/flows
POST   /v1/admin/flows/search
GET    /v1/admin/flows/{flow_id}
PATCH  /v1/admin/flows/{flow_id}
DELETE /v1/admin/flows/{flow_id}

# If Option B in P4 — first-class steps:
POST   /v1/admin/flows/{flow_id}/steps
GET    /v1/admin/flows/{flow_id}/steps
PATCH  /v1/admin/flow-steps/{step_id}
DELETE /v1/admin/flow-steps/{step_id}

POST   /v1/admin/flow-runs                       # body: { flow_id, submission_id }
GET    /v1/admin/flow-runs
POST   /v1/admin/flow-runs/search
GET    /v1/admin/flow-runs/{run_id}
POST   /v1/admin/flow-runs/{run_id}/advance      # action — manual advance in M1
POST   /v1/admin/flow-runs/{run_id}/cancel       # action — manual cancel
```

### URL surface (M5+)

Mostly the above, plus:

```
POST   /v1/admin/flows/{flow_id}/publish         # action
```

EventBridge consumers don't appear in the URL surface — they hook in via the Lambda's event-source configuration, separate from API Gateway routes.

### MCP tool surface (M1)

Single `/mcp` endpoint, CRUDLS naming, snake_case namespaces:

```
flows.{create, read, update, delete, list, search}
flow_runs.{create, read, update, delete, list, search}
flow_runs.advance                                 # action — manual advance
flow_runs.cancel                                  # action — manual cancel
```

Note `update`/`delete` on `flow_runs` may have limited semantics — FlowRuns are append-only state machines in M1; confirm during discuss whether they're omitted entirely or stubbed to error with `policy.invalid_state_transition`.

### MCP tool surface (M5+)

```
flows.publish                                     # when versioning lands
```

In M4, all of the above split into per-audience MCP servers per §8.4 §9.1.

### Naming discipline

- `flows` not `verify-flows` (domain vocabulary)
- `flow-runs` kebab-case in URLs; `flow_runs` snake_case as MCP namespace
- `flow-steps` kebab-case in URLs; `flow_steps` snake_case as MCP namespace (if first-class)
- ID prefixes: `flow_`, `run_`, `flowstep_` (or whatever's decided)

---

## 5. M1 acceptance criteria

When M1 is done, **both surfaces work**.

### REST (curl) — full lifecycle

```bash
# Setup — needs a Submission from Gather M1, OR a stub
# Assume Gather M1 has produced a submission with id sub_01HXX...

# Create a Flow with two trivial steps
FLOW=$(curl -X POST http://localhost:8001/v1/admin/flows \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d '{
      "workspace_id": "ws_01HXX...",
      "name": "Simple Two-Step",
      "definition": {
        "steps": [
          { "id": "step1", "type": "manual", "name": "Review" },
          { "id": "step2", "type": "manual", "name": "Approve" }
        ]
      }
    }' \
    | jq -r .id)

# Create a FlowRun against an existing Submission
RUN=$(curl -X POST http://localhost:8001/v1/admin/flow-runs \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d "{\"flow_id\": \"$FLOW\", \"submission_id\": \"sub_01HXX...\"}" \
    | jq -r .id)

# Advance to step2
curl -X POST http://localhost:8001/v1/admin/flow-runs/$RUN/advance \
    -H 'X-Debug-Principal-Id: au_dev_1'

# Advance past step2 → completed
curl -X POST http://localhost:8001/v1/admin/flow-runs/$RUN/advance \
    -H 'X-Debug-Principal-Id: au_dev_1'

# Verify completed status
curl http://localhost:8001/v1/admin/flow-runs/$RUN \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    | jq .status   # → "completed"
```

### MCP (Inspector) — full lifecycle

With `INSDI_DEBUG_PRINCIPAL_ID=au_dev_1`, connect MCP Inspector to `http://localhost:8001/mcp`:

1. `tools/list` enumerates `flows.{create,read,update,delete,list,search}`, `flow_runs.{create,read,list,search,advance,cancel}`
2. `flows.create` with the same `definition` payload → returns Flow response
3. `flow_runs.create` with `{ flow_id, submission_id }` → returns FlowRun with `status="running"`, `current_step_id="step1"`
4. `flow_runs.advance` with `{ id: run_id }` → returns FlowRun with `current_step_id="step2"`
5. `flow_runs.advance` again → returns FlowRun with `status="completed"`
6. `flow_runs.read` → returns the completed FlowRun
7. Error cases render correctly on both surfaces (e.g. advancing a completed run → `409 conflict.invalid_state_transition`)

### Parity check

Operations round-trip cleanly across surfaces (create Flow via REST, advance FlowRun via MCP, etc.).

Plus all §8.4 conventions visible in the responses on both surfaces.

---

## 6. Things to watch

- **Cross-service Submission reference:** A FlowRun's `submission_id` points at a Gather-owned row. In M1 you can stub a row in the local PG (no foreign-key enforcement across "services" — they share `public` schema), but the cross-service-read pattern (Verify reads Submission via SQL) is real from M5+ onwards.
- **No execution engine in M1.** The `advance` endpoint / `flow_runs.advance` tool exists as a manual placeholder. Resist the temptation to add even a tiny rules engine — that's M5+ work. Applies to both surfaces equally.
- **Flow definition JSON schema** — the shape of `definition` will evolve over many milestones as new step types appear. M1 should accept a permissive schema (a `steps` array) and let M5+ refine it.
- **EventBridge wiring** (consuming `submission.finalised`) is the substantive M5+ work. Until then Verify lives in isolation — humans manually create FlowRuns via REST or MCP.
- **Update/Delete semantics on FlowRuns:** A FlowRun isn't really updated through normal CRUD — its state changes through `advance` and `cancel`. M1 can either (a) omit `flow_runs.update`/`flow_runs.delete` from the MCP tool surface entirely, or (b) include them and have them error with `policy.invalid_state_transition`. Confirm during discuss.
