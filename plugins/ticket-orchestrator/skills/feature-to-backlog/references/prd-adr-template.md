# PRD and ADR template

## PRD structure

Use this structure unless a section is genuinely irrelevant:

```markdown
# <Project> — <Feature> PRD (add-on to <base PRD>)

| Item | Value |
|---|---|
| Product | ... |
| Owner | ... |
| Version / status | 1.0 — Ready for backlog refinement |
| Date | ... |
| Builds on | base PRD / architecture / implemented modules |
| Companion docs | ADR and Backlog filenames |
| Intended reader | coding agents and engineers |

## 1. How to use this document
## 2. Context and goals
### 2.1 Context
### 2.2 Goals for this release
### 2.3 Non-goals for this release
## 3. Decisions log
## 4. Roles, audiences and glossary
## 5+. Domain sections
## Visibility and permissions
## Workflow / lifecycle
## Screens and views
## Notifications and integrations
## Audit, privacy and retention
## Acceptance scenarios
## Hooks for future modules
## Open questions
```

### PRD requirements
- Assign stable requirement IDs to every testable requirement.
- Include configurable settings and seed defaults.
- State what happens on create, edit, approval, rejection, cancellation, deletion/archive, role changes and leavers when relevant.
- Define server-side visibility, list/search/filter inference protection, and what is omitted/masked.
- Define notifications by event and recipient.
- Include acceptance scenarios with concrete people/roles and dates/numbers where useful.
- If sensitive personal data is involved, explicitly define who sees values, how masked views behave, sensitive-view audit, retention and exports.

## ADR structure

```markdown
# <Project> — <Feature> ADR (add-on to <base ADR>)

| Item | Value |
|---|---|
| Product | ... |
| Builds on | implemented architecture |
| Implements | PRD filename and requirement families |

## 1. How to use this document
## 2. Summary of decisions
## 3. ADR-<X>01 — Module boundaries
## 4. ADR-<X>02 — Authorisation
## 5. ADR-<X>03 — Data model
## 6+. Domain-specific architecture decisions
## API summary
## Frontend
## Before implementation: checks against existing codebase
## Risks
```

### ADR requirements
- Extend, do not duplicate, implemented system primitives.
- Define data model with key tables/relationships/indexes and invariants.
- Define concurrency/idempotency/transaction behavior where workflows can race.
- Define authorisation enforcement and masking/projection, not just UI hiding.
- Define background jobs and reliable side effects.
- Define API surface at route/resource level.
- Include a “Before implementation” codebase inspection checklist so ticket implementation details can be grounded in reality.
