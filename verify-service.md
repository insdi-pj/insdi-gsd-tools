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

### M1 — Bare CRUD, no auth, conventions-correct

**Goal:** A FastAPI Lambda where a human can define a Flow with several Steps, manually create a FlowRun against a Submission, and advance the run through its steps. All under §8.4 conventions. No authentication.

**Phases (sketch — confirm in `/gsd-discuss-phase 1`):**

- **P1 — Scaffold:** Same template as Platform and Gather P1. The decisions on `_commons_pending/`, error envelope, pagination should follow whichever service goes first (Platform — if its conventions are settled, copy them).
- **P2 — Cross-service ORMs (read-only):** Define Gather's Submission ORM and Platform's Organisation/Workspace ORMs for reading. `_commons_pending/models/` — ask the user.
- **P3 — Flow CRUD:** Flow ORM with `flow_id`, `workspace_id`, `name`, `description`, `version` (always 1 in M1), `definition` (JSONB — the step graph), `created_at`, `updated_at`. Three-schema Pydantic. Routes: `POST /v1/admin/flows`, `GET /v1/admin/flows`, `POST /v1/admin/flows/search`, `GET /v1/admin/flows/{flow_id}`, `PATCH /v1/admin/flows/{flow_id}`, `DELETE /v1/admin/flows/{flow_id}`.
- **P4 — FlowStep CRUD:** Two design options to grill during discuss:
  - **Option A:** FlowSteps are embedded in the Flow's `definition` JSONB — no separate routes; you edit a Flow's steps by PATCHing the Flow
  - **Option B:** FlowSteps are first-class entities with their own routes (`POST /v1/admin/flows/{flow_id}/steps`, etc.)
  
  Option A is simpler and matches how most workflow tools represent step graphs. Option B is more API-surface-heavy but allows finer-grained access control later. **Recommend A for M1**, with option B reconsidered in M5+ if needed.

- **P5 — FlowRun creation:** `POST /v1/admin/flow-runs` accepts `{ flow_id, submission_id }`, creates a FlowRun row with `status=running`, `current_step_id=<first_step>`, `started_at`. **In M1 no actual step execution happens** — the FlowRun just exists in a state machine.
- **P6 — FlowRun advancement:** `POST /v1/admin/flow-runs/{run_id}/advance` — an admin manually advances the FlowRun to the next step. Updates `current_step_id`. If at the end, sets `status=completed`. This is a placeholder for real automated execution (M5+) but is enough to demonstrate the state machine works.
- **P7 — FlowRun listing/search:** `GET /v1/admin/flow-runs`, `POST /v1/admin/flow-runs/search`, `GET /v1/admin/flow-runs/{run_id}`. Filterable by `flow_id`, `submission_id`, `status`, `started_at`.
- **P8 — Polish.**

**Decisions to grill in `/gsd-discuss-phase 1`:**
- Embedded vs first-class FlowSteps (Option A vs B above)
- Flow `definition` JSONB shape — what does a step graph look like? A list of `{ id, type, config, next_step_id }`? A DAG with `dependencies`?
- Step types in M1 — given no real execution happens, "manual review" is enough; M5+ adds `integration_call`, `branch`, `calculate_run`, `gather_followup`
- FlowRun `status` enum — `running`, `completed`, `failed`, `cancelled`? Add `awaiting_input` and `paused`?
- Whether `current_step_id` is a denormalised field or computed from FlowRun's event history (M1 should denormalise — simpler)
- ID prefixes: `flow_`, `flowstep_` (if first-class), `run_`

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
- **MCP surface.**

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

### Naming discipline

- `flows` not `verify-flows` (domain vocabulary)
- `flow-runs` kebab-case
- `flow-steps` kebab-case (if first-class)
- ID prefixes: `flow_`, `run_`, `flowstep_` (or whatever's decided)

---

## 5. M1 acceptance criteria

A human can run this script:

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

Plus all §8.4 conventions visible in the responses.

---

## 6. Things to watch

- **Cross-service Submission reference:** A FlowRun's `submission_id` points at a Gather-owned row. In M1 you can stub a row in the local PG (no foreign-key enforcement across "services" — they share `public` schema), but the cross-service-read pattern (Verify reads Submission via SQL) is real from M5+ onwards.
- **No execution engine in M1.** The `advance` endpoint exists as a manual placeholder. Resist the temptation to add even a tiny rules engine — that's M5+ work.
- **Flow definition JSON schema** — the shape of `definition` will evolve over many milestones as new step types appear. M1 should accept a permissive schema (a `steps` array) and let M5+ refine it.
- **EventBridge wiring** (consuming `submission.finalised`) is the substantive M5+ work. Until then Verify lives in isolation — humans manually create FlowRuns from the admin API.
