# insdi GSD Project Briefs

This folder contains the GSD project briefs for the four insdi service Lambdas. Each brief is the **starting context** you paste into `/gsd-new-project` for that service's repo. The shared files in `_shared/` capture the cross-cutting methodology and conventions so each brief stays focused on the service-specific shape.

## Files

```
gsd-briefs/
├── _shared/
│   ├── INSDI_GSD_PRINCIPLES.md         ← How to structure milestones and phases (layered complexity + MCP parity)
│   ├── INSDI_API_CONVENTIONS.md        ← §8.4 conventions distilled (REST + MCP)
│   └── INSDI_COMMONS_PROTOCOL.md       ← How to handle insdi-commons references
├── platform-service.md                 ← Trust kernel: identity, auth, Orgs, Workspaces, etc.
├── gather-service.md                   ← Data collection: Templates, Links, Submissions
├── verify-service.md                   ← Workflow: Flows, FlowSteps, FlowRuns
└── calculate-service.md                ← Computation: Workbooks, Sets, Tests, CalculateRuns
```

## MCP from day one

insdi is an AI-native platform — its API is *equally* a contract for humans (REST) and for AI agents (MCP). §8.4 §1.4 and §9.3 mandate that REST and MCP are "two surfaces over one state machine," structurally enforced by co-locating both in the same service Lambda. The briefs reflect this: **every M1 phase that adds a REST route ships its matching MCP tool in the same PR**.

What this means concretely:
- M1 ships both surfaces with no auth — REST uses `X-Debug-Principal-Id`, MCP uses `INSDI_DEBUG_PRINCIPAL_ID` env var, both resolve to the same `UserContext`
- M2 wires Cognito auth onto both surfaces — same `UserContext`, same use-case functions, surface-adapted auth wire formats
- M3 wires audit emission once (in use-case functions); both surfaces emit identical audit events
- M4 introduces MCP-specific production architecture (per-audience server split, OAuth at session, `.well-known/mcp`, build-time manifest assembly)

MCP tools follow CRUDLS naming: `<resource>.{create, read, update, delete, list, search}`. Note `read` (not `get`). Resources use snake_case in MCP namespaces (`flow_runs`, not `flow-runs`); REST paths remain kebab-case.

Local MCP testing uses MCP Inspector: `npx @modelcontextprotocol/inspector http://localhost:8000/mcp`. Every M1 acceptance criteria has both a curl section and an Inspector section.

## Typed content from day one — the AI usability principle

A corollary of MCP parity that comes up across the briefs: **substantive content fields use typed Pydantic models from `insdi-commons`, never `dict[str, Any]`**.

When an admin tells an AI client *"create a patient intake form with name, DOB, allergies, and a consent checkbox"*, the AI calls `templates.create`. If the Template's `schema` field is typed as `dict`, the AI sees only `{"type": "object"}` in the MCP `inputSchema` and must guess at the inner shape — producing inconsistent results and breaking the MCP-as-AI-contract promise. With a typed `commons.schemas.TemplateSchema`, the AI sees allowed field types as `Literal` enums, discriminated unions for type-specific constraints, and required vs optional sub-fields. It produces structurally-valid input on the first try.

Per PJ's direction, most entity-content Pydantic models are already defined in `insdi-commons` and are referenced from M1 P1. This applies to:
- Template `schema` (Gather)
- Flow `definition` (Verify)
- Workbook `definition` (Calculate)
- `submitter_allowlist`, `auth_required_floor` (Platform; used cross-service via the policy chain)

See `_shared/INSDI_GSD_PRINCIPLES.md` §4 for the full discipline.

## The drafting-tool progression (M5+, per service)

Typed content models make the `<resource>.create` family AI-usable. A higher-value AI pattern comes later: **natural-language description → fully-formed input**. Each service eventually grows a drafting tool:

- `templates.draft_from_description` (Gather, M5 or M6)
- `flows.draft_from_description` (Verify, M6+)
- `workbooks.draft_from_description` (Calculate, M6+)
- `organisations.draft_policies_from_description` (Platform, M6+)

These are backed by insdi-controlled LLM calls with curated prompts and few-shot examples. The admin describes intent, the tool proposes a structured input with rationale, the admin reviews and confirms before calling `.create`. **They land M5+ — never in M1.** Drafting tools require the structured surface to be proven, the typed content models to be stable, and a corpus of successful production inputs to draw few-shot examples from. Premature drafting tools produce inconsistent output that erodes trust in the platform.

## How to use

For each service, in its dedicated repo:

1. Run `/gsd-new-project` in the empty (or scaffold-only) repo.
2. When GSD asks for the vision/scope, paste in:
   - The contents of `_shared/INSDI_GSD_PRINCIPLES.md`
   - The contents of `_shared/INSDI_API_CONVENTIONS.md`
   - The contents of `_shared/INSDI_COMMONS_PROTOCOL.md`
   - The contents of the relevant service brief (e.g. `platform-service.md`)
3. Answer GSD's clarifying questions, referring back to the brief.
4. GSD produces `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md`.
5. Run `/gsd-discuss-phase 1` to lock M1 P1 decisions. The brief calls out specific decisions to grill on during each milestone's discuss phase.
6. Run `/gsd-plan-phase 1`, `/gsd-execute-phase 1`, `/gsd-verify-work 1`, `/gsd-ship 1`.
7. Repeat for each phase, milestone.

## Build order

The four services are independent GSD projects but have soft dependencies:

```
platform-service (M1)  ──┬──→  gather-service (M1)
                         │
                         ├──→  verify-service (M1)  (also reads gather)
                         │
                         └──→  calculate-service (M1)  (also reads gather)
```

**Recommended build order:** platform-service M1 first. Once Platform's Organisation and Workspace ORMs exist and migrations are applied to the shared schema, the other three services have what they need to read those tables.

Alternative: build all four services' M1s in parallel, stubbing cross-service tables locally during P2 of each. When migrating to a real shared schema in M4, the stubs go away. This is faster end-to-end but means each service's M1 has temporary scaffolding to remove later.

## Out of scope (explicitly)

§8.2 specifies **six** service Lambdas — Platform, Resources, Gather, Verify, Calculate, Audit. The current scope is **four** (Platform, Gather, Verify, Calculate). Consequences:

- **Resources Lambda absent:** File attachments on Submissions and Triggers on entity events are deferred. Where the briefs reference Files or Triggers, they're flagged as M5+ work pending Resources.
- **Audit Lambda absent:** M3 wires audit emission into the PG outbox tables, but no drainer Lambda exists to move them to DynamoDB. Audit events accumulate durably in PG — acceptable per §8.2 §9, but ROADMAP entries call this out so a future audit-service build picks up cleanly.
- **EventBridge cross-service workflows** rely on producers and consumers existing in different services. With four services only, some flows are partial: Gather can publish `submission.finalised` (M5+) but Resources can't react with Trigger evaluation (out of scope). Verify can consume `submission.finalised` once Verify M5+ ships. Plan accordingly.

## The meta-question — how this discipline applies to future GSD builds

The methodology in `_shared/INSDI_GSD_PRINCIPLES.md` is general — it isn't insdi-specific in its structure (the philosophy: M1 is bare-functionality conventions-correct; auth is M2; audit is M3; production hardening is M4; advanced features M5+). The specifics of which conventions apply (§8.4) and which symbols come from `insdi-commons` are insdi-specific.

To apply this discipline to any future insdi GSD project:

1. Reference `_shared/INSDI_GSD_PRINCIPLES.md` and `_shared/INSDI_API_CONVENTIONS.md` and `_shared/INSDI_COMMONS_PROTOCOL.md` at the start of the new GSD project (paste them in or import them as context).
2. Write a brief for the new service following the same template (vision, entities owned, milestone roadmap, URL surface, M1 acceptance criteria, things to watch).
3. The four service briefs in this folder are reference templates.

This is the answer to "how do we include this kind of instruction in all GSD builds": a single shared methodology doc, referenced by every per-service brief. Not a custom skill, not a fork of GSD's own commands — just a discipline encoded in markdown and applied consistently. Per PJ's preference, the discipline lives outside Notion (this folder lives in version control alongside the briefs and is the canonical project rule).
