# GSD Project Brief — `insdi-calculate-service`

**Lambda:** `insdi-calculate-service`
**Domain:** Computation — Workbooks, Sets, Tests, CalculateRuns
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

`insdi-calculate-service` is the computation engine. A **Workbook** is a versioned definition of a calculation (think: spreadsheet-like rules, scoring models, eligibility tests, financial projections). A **Set** is a collection of input cases. A **Test** validates Workbook behaviour against known inputs/outputs. A **CalculateRun** is one execution of a Workbook against an input — typically a Gather Submission.

Calculate sits downstream of Gather and alongside Verify: a finalised Submission can trigger a CalculateRun, and a Verify FlowStep can bind a CalculateRun as input. CalculateRun outputs feed back into Verify decisions (where bound) and into Resources Triggers.

In v1 scope: REST API surface for designing Workbooks and their Sets/Tests, manually invoking CalculateRuns, reading results.

Calculate is conceptually the simplest of the three service-family Lambdas in M1 (Gather, Verify, Calculate) because there's no in-flight state machine (like Gather sessions) or workflow orchestration (like Verify Flows) — just "given inputs, produce outputs."

---

## 2. Entities owned (from §8.2 §2)

| Entity | Storage | Notes |
|---|---|---|
| `Workbook` (versioned) | PG | The calculation definition |
| `Set` | PG | Named collection of input cases |
| `Test` | PG | Input/expected-output pair for a Workbook |
| `CalculateRun` | PG | One execution of a Workbook against input |

**Cross-service reads (direct SQL):**

- `Organisation`, `Workspace` (Platform-owned)
- `Submission` (Gather-owned) — a CalculateRun's input

---

## 3. Milestone roadmap

### M1 — Bare CRUDLS, no auth, conventions-correct, MCP parity

**Goal:** A FastAPI Lambda where a human can define a trivial Workbook (a simple formula like "score = field_a * 2 + field_b"), create a Set of test cases, define Tests, and trigger a CalculateRun against a Submission to get a result — on **both** REST and MCP surfaces. All under §8.4 conventions, no authentication.

**Phases (sketch — confirm in `/gsd-discuss-phase 1`):**

Each entity-phase delivers REST routes AND matching MCP tools in the same PR per principles doc §3.

- **P1 — Scaffold:** Same shape as the others — repo layout including `mcp/` directory, SAM, FastAPI with MCP server mounted at `/mcp` via streamable HTTP, Postgres locally, Alembic, structured logger, error envelope, pagination, filter parser. Reuse decisions from Platform's M1 P1 (MCP library, conventions, `_commons_pending/` shape). Smoke-test: MCP Inspector connects and lists 0 tools.

- **P2 — Cross-service ORMs (read-only):** Submission ORM (Gather-owned), Organisation/Workspace ORMs (Platform-owned). `_commons_pending/` — ask user. No tools.

- **P3 — Workbook CRUDLS:**
  - Three-schema Pydantic. Workbook ORM with `workbook_id`, `workspace_id`, `name`, `description`, `version` (always 1 in M1), `definition` (JSONB — computation rules), `created_at`, `updated_at`.
  - Use-case functions in `services/workbooks.py`: `create_workbook`, `read_workbook`, `update_workbook`, `delete_workbook`, `list_workbooks`, `search_workbooks`.
  - REST routes: `POST /v1/admin/workbooks`, `GET`, `GET /{id}`, `PATCH /{id}`, `DELETE /{id}`, `POST /search`.
  - MCP tools: `workbooks.{create, read, update, delete, list, search}`.

- **P4 — Set CRUDLS:** Set is a named collection of input cases.
  - REST routes: `GET /v1/admin/workbooks/{wb_id}/sets` (nested enumeration), `POST /v1/admin/sets`, `GET /v1/admin/sets/{set_id}`, `PATCH /v1/admin/sets/{set_id}`, `DELETE /v1/admin/sets/{set_id}`.
  - MCP tools: `sets.{create, read, update, delete, list}`. Listing accepts an optional `workbook_id` filter parameter — the URL nesting maps to a tool input parameter, not a separate tool.
  - **Confirm during discuss whether Sets nest under Workbook or are top-level with FK** — §8.4 §3.3 guidance: child enumeration nests, child with own ID flat. Recommend flat with FK + nested REST enumeration. MCP tools are always flat with optional parent-FK filter.

- **P5 — Test CRUDLS:** Test is `{ inputs, expected_outputs }` for a specific Workbook. Same pattern as Set.
  - REST: `GET /v1/admin/workbooks/{wb_id}/tests`, `POST /v1/admin/tests`, `GET /v1/admin/tests/{test_id}`, `PATCH /v1/admin/tests/{test_id}`, `DELETE /v1/admin/tests/{test_id}`.
  - MCP tools: `tests.{create, read, update, delete, list}`.

- **P6 — Workbook execution stub:** A `services/compute.py` use-case function `evaluate_workbook(workbook_id, input)` that evaluates a Workbook's `definition` against an input dict. **In M1, support only the simplest possible computation model:**
  - A `definition` like `{ "outputs": { "score": "field_a * 2 + field_b" } }`
  - A safe expression evaluator (e.g. asteval, simpleeval — not raw `eval()` — confirm in discuss)
  - Returns `{ "outputs": { "score": 42 } }`
  - No conditional logic, no aggregation, no per-row processing
  - More complex computation is M5+
  - This is the use-case function called by both `calculate_runs.create` (P7) and `tests.run` (P9). No tool of its own; it's internal.

- **P7 — CalculateRun creation + execution:**
  - REST: `POST /v1/admin/calculate-runs` accepts `{ workbook_id, input }` or `{ workbook_id, submission_id }`, reads the input, invokes the executor synchronously, persists a CalculateRun row with `status`, `outputs`, `started_at`, `completed_at`, `error` (if any). Returns the CalculateRun.
  - MCP: `calculate_runs.create` tool with the same input shape and response. **Synchronous execution in M1** (the Lambda blocks until done). Async + run-status-polling is M5+.

- **P8 — CalculateRun listing/read/search:**
  - REST: `GET /v1/admin/calculate-runs`, `POST /v1/admin/calculate-runs/search`, `GET /v1/admin/calculate-runs/{run_id}`. Filterable by `workbook_id`, `submission_id`, `status`.
  - MCP: `calculate_runs.{read, list, search}`. **No update/delete on CalculateRun** — runs are immutable execution records.

- **P9 — Test execution:**
  - REST: `POST /v1/admin/tests/{test_id}/run` — executes the Test's input against its bound Workbook, compares actual outputs to expected, returns pass/fail with diff.
  - MCP: `tests.run` tool taking `{ id: test_id }` and returning the same pass/fail result.
  - Tool description: "Execute the Test against its bound Workbook and compare actual vs expected outputs. Returns `{ passed: bool, actual_outputs, expected_outputs, diff }`. Used for validating Workbook behaviour."

- **P10 — Polish:** Manual test script for REST (curl) and MCP (Inspector session). OpenAPI + `tools/list` sanity checks.

**Decisions to grill in `/gsd-discuss-phase 1`:**
- **Confirm the Workbook `definition` field uses the typed `insdi-commons` model, not `dict`.** Per principles doc §4, the WorkbookDefinition shape comes from `commons.schemas.WorkbookDefinition` — so MCP's `inputSchema` for `workbooks.create` carries the full structure (allowed expression types, output declarations, intermediate variables if supported) and AI clients can produce valid input on the first try. If commons' current shape is too minimal for what you want M1 to support, schedule a commons extension — don't shadow locally with `dict`.
- Workbook `definition` *content* coverage in M1 — what's the smallest supported subset? A flat `{ outputs: { name: expression } }` (no intermediates, no conditionals)? Confirm what commons currently supports vs what M1 needs
- Set's `inputs` and Test's `expected_outputs` — also typed via commons (likely a `dict[str, JsonValue]` keyed by field id is appropriate here, but the keys should be constrained by the bound Workbook's inputs; check if commons has a typed shape for this)
- Expression evaluator library — `asteval`, `simpleeval`, custom? Security matters — never use raw `eval`
- Set: nested-only or flat-with-FK
- Sync vs async CalculateRun in M1 — strongly recommend sync (Lambda timeout permitting). Async needs a Step Functions or worker pattern; M5+ work
- Whether CalculateRun input is `Submission` (read via FK) or direct `input` payload, or both. Probably both — submitting a raw payload is the developer's iteration loop; submitting via `submission_id` is the production path
- ID prefixes: `wb_`, `set_`, `test_`, `crun_` or `calc_`
- MCP library — must match Platform's choice

**Out of scope for M1:**
- JWT validation, auth (M2)
- Audit (M3)
- Idempotency, ETag, rate-limit (M2)
- EventBridge consumption (`submission.finalised` auto-trigger) — M5+
- EventBridge publication (`calculate.completed`) — M5+
- Workbook versioning / publish flow — M5+
- Complex computation (conditionals, aggregation, multi-step, lookups) — M5+
- Async execution — M5+
- Workbook → Verify FlowStep binding — M5+ (in Verify, reads Calculate)
- MCP-specific production concerns (per-audience server split, OAuth, discovery) — M4

---

### M2 — Cognito JWT validation, real auth

- **P1 — `RequireAdmin` on every route.**
- **P2 — Workspace-membership filtering.**
- **P3 — Idempotency-Key on creates and execution actions.**
- **P4 — ETag + If-Match:** ETag on Workbook GETs. `If-Match` required on PATCH/DELETE of Workbook (§8.4 §6.6).
- **P5 — Rate limiting.**
- **P6 — `Sunset` / `Deprecation` middleware.**

---

### M3 — Audit emission

- **P1 — `emit_with_tx` on writes** (workbook.created, calculate_run.created, etc.). Tier A.
- **P2 — Tier C on reads.**
- **P3 — `X-Request-Id`.**

---

### M4 — Production hardening

Same shape as the others.

---

### M5+ — Advanced Calculate features

- **EventBridge consumption:** Subscribe to `submission.finalised`; if the Template has a bound Workbook, auto-create a CalculateRun. Idempotent on event_id.
- **EventBridge publication:** `calculate.completed`, `calculate.failed`.
- **Async execution:** Long-running Workbooks return 202 with a polling URL. Step Functions or SQS-backed worker pattern.
- **Workbook versioning / publish flow:** Like Templates and Flows.
- **Complex computation model:**
  - Intermediate variables / cells
  - Conditional branching (`if x then y else z`)
  - Aggregation (sum, count, avg over a list)
  - Lookups (against Sets, against external data)
  - Per-row processing (vectorised computation over arrays)
  - User-defined functions or modules
- **Sandboxing / resource limits:** CPU time, memory, allowed operations.
- **Diff-on-Test:** Better Test outcome rendering with structured diffs.
- **`workbooks.draft_from_description` (M6+, post-stability):** A natural-language → WorkbookDefinition drafting tool. Admin says *"score a loan application based on income, credit history, and debt-to-income ratio"*; the tool returns a proposed `WorkbookCreateInput` with rationale and alternatives. Backed by insdi-controlled LLM call. **Requires:** typed `WorkbookDefinition` proven in production with conditionals and intermediates, a corpus of successful Workbooks for few-shot examples, evaluation harness (the drafting tool itself produces Workbooks; Tests are the natural evaluation mechanism — "draft a Workbook from this description, then check it passes these Tests"). See `INSDI_GSD_PRINCIPLES.md` §4 for the full progression rationale.
- **`workbooks.suggest_tests` (M6+):** Given an existing Workbook, propose Tests that meaningfully cover its branches. Pairs naturally with the drafting tool.
- **MCP enhancements (M5+ adds new tools as features land):** Each M5+ feature ships its REST routes and matching MCP tools together. `workbooks.publish` (when versioning lands), `calculate_runs.cancel` and `calculate_runs.read_status` (when async lands), etc. The MCP surface grows alongside REST — never as a separate workstream.

---

## 4. Key §8.4 conventions

### URL surface (M1)

```
POST   /v1/admin/workbooks
GET    /v1/admin/workbooks
POST   /v1/admin/workbooks/search
GET    /v1/admin/workbooks/{wb_id}
PATCH  /v1/admin/workbooks/{wb_id}
DELETE /v1/admin/workbooks/{wb_id}

GET    /v1/admin/workbooks/{wb_id}/sets          # nested enumeration
POST   /v1/admin/sets                            # body: { workbook_id, ... }
GET    /v1/admin/sets/{set_id}
PATCH  /v1/admin/sets/{set_id}
DELETE /v1/admin/sets/{set_id}

GET    /v1/admin/workbooks/{wb_id}/tests
POST   /v1/admin/tests
GET    /v1/admin/tests/{test_id}
PATCH  /v1/admin/tests/{test_id}
DELETE /v1/admin/tests/{test_id}
POST   /v1/admin/tests/{test_id}/run             # action

POST   /v1/admin/calculate-runs                  # body: { workbook_id, input } or { workbook_id, submission_id }
GET    /v1/admin/calculate-runs
POST   /v1/admin/calculate-runs/search
GET    /v1/admin/calculate-runs/{run_id}
```

### URL surface (M5+)

```
POST   /v1/admin/workbooks/{wb_id}/publish       # action — create new version
POST   /v1/admin/calculate-runs/{run_id}/cancel  # action — cancel async run
GET    /v1/admin/calculate-runs/{run_id}/status  # for async polling
```

### MCP tool surface (M1)

Single `/mcp` endpoint, CRUDLS naming, snake_case namespaces:

```
workbooks.{create, read, update, delete, list, search}
sets.{create, read, update, delete, list}
tests.{create, read, update, delete, list}
tests.run                                         # action
calculate_runs.create                             # synchronous execution in M1
calculate_runs.{read, list, search}               # no update/delete — runs are immutable
```

### MCP tool surface (M5+)

```
workbooks.publish                                 # when versioning lands
calculate_runs.cancel                             # when async lands
calculate_runs.read_status                        # when async lands
```

In M4, all of the above split into per-audience MCP servers per §8.4 §9.1.

### Naming

- `workbooks` (domain vocabulary)
- `calculate-runs` kebab-case
- `sets`, `tests` (simple plurals)
- ID prefixes: `wb_`, `set_`, `test_`, `crun_` (confirm)

---

## 5. M1 acceptance criteria

When M1 is done, **both surfaces work**.

### REST (curl) — full lifecycle

```bash
# Create a Workbook with a trivial computation
WB=$(curl -X POST http://localhost:8002/v1/admin/workbooks \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d '{
      "workspace_id": "ws_01HXX...",
      "name": "Simple Scorer",
      "definition": {
        "outputs": {
          "score": "field_a * 2 + field_b"
        }
      }
    }' \
    | jq -r .id)

# Execute it against a raw input
curl -X POST http://localhost:8002/v1/admin/calculate-runs \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d "{\"workbook_id\": \"$WB\", \"input\": {\"field_a\": 5, \"field_b\": 3}}" \
    | jq

# Expected: status="completed", outputs={"score": 13}

# Execute against a Submission (presuming Gather M1 has produced one)
curl -X POST http://localhost:8002/v1/admin/calculate-runs \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d "{\"workbook_id\": \"$WB\", \"submission_id\": \"sub_01HXX...\"}" \
    | jq

# Create a Test and run it
TEST=$(curl -X POST http://localhost:8002/v1/admin/tests \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    -H 'Content-Type: application/json' \
    -d "{
      \"workbook_id\": \"$WB\",
      \"name\": \"basic-case\",
      \"input\": {\"field_a\": 10, \"field_b\": 1},
      \"expected_outputs\": {\"score\": 21}
    }" \
    | jq -r .id)

curl -X POST http://localhost:8002/v1/admin/tests/$TEST/run \
    -H 'X-Debug-Principal-Id: au_dev_1' \
    | jq
# Expected: { "passed": true, "actual_outputs": {"score": 21}, ... }
```

### MCP (Inspector) — full lifecycle

With `INSDI_DEBUG_PRINCIPAL_ID=au_dev_1`, connect MCP Inspector to `http://localhost:8002/mcp`:

1. `tools/list` enumerates: `workbooks.{create,read,update,delete,list,search}`, `sets.{create,read,update,delete,list}`, `tests.{create,read,update,delete,list}`, `tests.run`, `calculate_runs.{create,read,list,search}`
2. Call `workbooks.create` with the same payload as the curl script above → returns Workbook response
3. Call `calculate_runs.create` with `{ workbook_id: WB, input: {field_a: 5, field_b: 3} }` → returns CalculateRun with `status="completed"`, `outputs={"score": 13}`
4. Call `calculate_runs.create` with `{ workbook_id: WB, submission_id: ... }` → executes against the submission's body
5. Call `tests.create` and then `tests.run` → returns `{ passed: true, actual_outputs, expected_outputs, diff }`
6. Error rendering: `calculate_runs.create` with a workbook that has invalid syntax in its definition → returns `isError: true` with a `validation.invalid_workbook_definition` (or similar) problem+json body
7. `calculate_runs.list` with cursor pagination works correctly

### Parity check

A Workbook created via REST is executable via MCP and vice versa. Tests defined via either surface run identically. Same `outputs` returned regardless of which surface invoked.

Plus all §8.4 conventions visible in the responses on both surfaces.

---

## 6. Things to watch

- **Expression evaluator security:** Use a safe library; never `eval`. Whatever's chosen in M1 P6 should be sandboxed enough that a malicious Workbook can't break out. M5+ may add stricter sandboxing. This applies equally regardless of whether the Workbook was created/executed via REST or MCP.
- **Lambda timeout:** Synchronous execution in M1 means Workbook complexity is bounded by the Lambda timeout (15 minutes max, default much lower). A trivial scorer is fine; complex Workbooks need M5+ async. MCP tool calls have the same timeout — they're served by the same Lambda.
- **Submission read pattern:** Same as Verify — Calculate reads Gather's Submission table via direct SQL on the shared schema. The Submission ORM lives in `_commons_pending/`.
- **The verb naming carve-out:** `POST /v1/admin/tests/{test_id}/run` is an action — verb after the noun-shaped sub-path per §8.4 §1.3 (`Actions are POST to noun-shaped sub-paths, not verbs in the path`). The action's *name* (`run`) is a verb; the path *shape* (`/tests/{id}/run`) is still noun-then-action. This is consistent with `templates/{id}/publish` and `flow-runs/{id}/advance`. The MCP equivalent (`tests.run`) follows the `noun.verb` ordering — same convention, surface-appropriate syntax.
- **CRUDLS vs computed-action distinction:** `tests.run` is *not* a CRUDLS member — it's a domain action. CRUDLS-named tools (`tests.read`, etc.) do plain CRUD. Actions get their own tool names (`tests.run`, `workbooks.publish`). Don't conflate.
