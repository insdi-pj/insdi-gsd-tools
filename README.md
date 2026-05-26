# insdi GSD Project Briefs

This folder contains the GSD project briefs for the four insdi service Lambdas. Each brief is the **starting context** you paste into `/gsd-new-project` for that service's repo. The shared files in `_shared/` capture the cross-cutting methodology and conventions so each brief stays focused on the service-specific shape.

## Files

```
gsd-briefs/
├── _shared/
│   ├── INSDI_GSD_PRINCIPLES.md         ← How to structure milestones and phases (layered complexity)
│   ├── INSDI_API_CONVENTIONS.md        ← §8.4 conventions distilled to a quick lookup
│   └── INSDI_COMMONS_PROTOCOL.md       ← How to handle insdi-commons references
├── platform-service.md                 ← Trust kernel: identity, auth, Orgs, Workspaces, etc.
├── gather-service.md                   ← Data collection: Templates, Links, Submissions
├── verify-service.md                   ← Workflow: Flows, FlowSteps, FlowRuns
└── calculate-service.md                ← Computation: Workbooks, Sets, Tests, CalculateRuns
```

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
