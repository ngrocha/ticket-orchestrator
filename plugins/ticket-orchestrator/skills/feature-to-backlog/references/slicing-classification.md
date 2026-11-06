# Slicing, ticket classification and execution strategy

This reference defines the planning-time classification contract. Apply it to every ticket before the backlog is handed to implementation. Classification is a structured planning judgement, not a promise of elapsed time or a substitute for repository inspection.

## 1. Vertical slice rule

A ticket should normally be independently demoable and testable. Include all layers required for that capability:

`data → domain → authorisation → API → minimal UI → tests`

Examples:
- Good: “Employee submits a vacation request and sees Pending in their calendar.” This includes the data mutation, validation, authz, route, request form/calendar change and tests.
- Bad: “Create absence database tables.” If schema work is cross-cutting and must precede slices, keep the foundation ticket as small as possible and identify exactly which first slice it unlocks.
- Bad: “Build absence frontend.” It is horizontal and too broad.

A foundation ticket is acceptable only when the capability genuinely cannot be delivered as a vertical slice. State why, what it unlocks, and why the work cannot be included in the first consumer slice.

## 2. Phase rules

- Each phase has a user-observable or engineering exit criterion.
- Put prerequisite foundations in the earliest phase, but avoid a long horizontal setup phase.
- Dependencies should normally point to an earlier phase. Same-phase dependencies are allowed when sequencing is unavoidable.
- Do not make a ticket depend on a later phase.
- Separate later/polish scope instead of blocking the core usable flow.
- Phase ticket numbers are project-specific. If no convention is supplied, use sequential `PH-001`, `PH-002`, ... and phases 1..N.

## 3. Required classification for every ticket

Every ticket MUST include:

- **Complexity:** XS / S / M / L / XL.
- **Complexity score:** total and dimension scores, using the scorecard below (for example `11/20: scope 2, logic 3, integration/data 2, verification 2, uncertainty 2`).
- **Risk:** Low / Medium / High.
- **Change type:** Bug / Feature / Refactoring / Infrastructure.
- **Modules affected:** integer count plus names when known; use `Unknown` rather than inventing names.
- **Acceptance criteria:** integer count.
- **Architectural impact:** None / Local / Cross-cutting / New architectural decision.
- **Expected verification effort:** Low / Medium / High, with the important test categories and environments.
- **Classification rationale:** one to three concise sentences explaining the score, risk and any override.
- **Confidence:** High / Medium / Low.
- **Classification basis:** PRD/ADR only, repository inspection, supplied architecture/docs, or a combination. Name the actual evidence where possible.
- **Execution strategy:** planner, implementer, reviewer model recommendations and any required specialist/human gate.

If the repository has not been inspected, do not imply that exact modules, paths, test commands or implementation patterns have been verified. Mark those details as expected/unknown and reduce confidence when that uncertainty could change the classification.

## 4. Complexity scorecard

Score each dimension from 1 to 4. Use the highest description that materially applies; do not award a low score simply because details are missing.

| Dimension | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| **Scope** | One small, localized change | One vertical slice across 1–2 modules | Several layers or 3–5 modules | Broad/cross-domain scope or more than 5 modules |
| **Logic and state** | Straightforward mapping/presentation | Conventional validation or simple state transition | Non-trivial business rules, permissions or multi-step workflow | Complex state machine, concurrency, ordering, reconciliation or subtle invariants |
| **Integration and data** | No new integration or data shape | Existing API/schema/pattern reused | New/changed API, schema, job or one external integration | Multiple integrations, migration/backfill, data movement or compatibility concerns |
| **Verification burden** | Focused unit/component tests | Unit plus API/integration or one UI path | Multiple layers, roles, environments or meaningful regression surface | Security/leak, concurrency, migration, resilience or broad end-to-end verification |
| **Uncertainty** | Fully specified; established pattern | Minor assumptions; pattern is known | Important behavior or implementation details need investigation | Architecture/product decision unresolved or unfamiliar technology/domain |

**Total score:** add the five dimension scores (minimum 5, maximum 20).

| Total | Initial complexity |
|---:|---|
| 5–6 | XS |
| 7–9 | S |
| 10–13 | M |
| 14–17 | L |
| 18–20 | XL |

### Complexity guardrails and overrides

The total is a consistency aid, not an averaging mechanism that can hide a critical dimension:

- Any dimension scored **4** makes the ticket at least **M**.
- Logic/state = 4, integration/data = 4, or verification = 4 normally makes it at least **L**. If that concern can be isolated into a smaller independently testable slice, split it and score the resulting tickets instead.
- A ticket spanning multiple independently demoable capabilities must be split even if its score is below 18.
- **XL** means the work is not suitable for one normal implementation ticket. Split it into smaller slices. Retain XL only when a documented atomicity constraint makes splitting unsafe or impossible; explain that constraint and require explicit human planning approval before implementation.
- Do not lower complexity because an agent is expected to implement the ticket. Score the work, not the model.
- Risk is independent of complexity. A small authorization change can be XS/S and still High risk.

The ticket's displayed label must agree with the score and all applicable guardrails. If an override changes the label, retain the raw total and explain the override.

## 5. Risk classification

Risk measures the consequence of a defect, not the amount of work.

- **High:** authorization/privacy/security, destructive migrations, money/payroll/legal correctness, concurrency that can corrupt balances/state, external migrations, irreversible data changes, or critical availability/reliability paths.
- **Medium:** workflow/state changes, background jobs, new integrations, non-trivial migrations, complex UI interaction, or changes with a meaningful but recoverable user impact.
- **Low:** isolated presentation/configuration or straightforward additive behavior with strong existing patterns and limited blast radius.

When multiple categories apply, choose the highest risk. Explain the concrete failure mode. High risk must have explicit negative/failure-path tests and a named review gate; security/privacy work requires an authorization and data-exposure review even if the change is small.

## 6. Verification effort

- **High:** unit + API/integration + relevant concurrency/security/leak/migration/resilience tests + deployed E2E or multiple environments, as applicable.
- **Medium:** unit + API/integration + one UI/E2E path, as applicable.
- **Low:** focused unit or component tests plus the existing regression suite.

Name the actual suites/categories that apply. Do not write only “test thoroughly”. If the repository is unknown, describe the required test types rather than inventing command names. Verification effort should be consistent with risk and the verification-burden score; explain any apparent mismatch.

## 7. Confidence and evidence

Confidence describes how reliable the classification is given the available evidence:

- **High:** requirements and architecture are clear, relevant repository areas/patterns were inspected, and scope/dependencies are understood.
- **Medium:** requirements are mostly clear but one or more implementation details, module boundaries or test approaches remain assumptions.
- **Low:** important behavior, architecture, dependencies or repository constraints are unknown; classification may change after discovery.

Record the basis explicitly, for example:
- `PRD + ADR + repository inspection (modules A/B and existing authorization middleware)`
- `PRD + ADR only; repository not supplied`
- `Feature brief only; data lifecycle and integration contract unresolved`

Low confidence is a planning flag, not a reason to invent precision. Add a discovery task or an explicit planning checkpoint when the uncertainty could change scope, risk, sequencing or architecture. Do not mark implementation-ready if a blocking decision remains unresolved.

## 8. Execution strategy recommendation

Recommend a role-based model handoff for each ticket. This is a starting policy for a Claude Code-style workflow; actual model availability, team policy, cost limits and tool permissions take precedence. Keep the implementer bounded by an explicit file scope, acceptance criteria and verification instructions. The orchestrator/reviewer must inspect the actual diff and verification evidence independently.

| Ticket profile | Planner | Implementer | Reviewer | Notes |
|---|---|---|---|---|
| XS/S, Low risk, established pattern | Sonnet | Haiku | Sonnet | Use a concise plan. Sonnet can implement instead when the task needs more judgement than mechanical execution. |
| M, Medium risk or non-trivial logic | Sonnet; Opus if uncertainty or design complexity is material | Haiku for bounded work; Sonnet if implementation requires substantial reasoning | Sonnet; Opus for material architectural or high-impact review | Prefer a written implementation brief before delegation. |
| L, Medium/High risk | Opus | Haiku only when decomposed into bounded, well-specified work; otherwise Sonnet | Opus | Require explicit verification plan and independent diff review. Split L further when practical. |
| XL | Opus for decomposition | No single-ticket implementation until split; then route each child ticket | Opus for decomposition review; route child reviews by risk | Keep XL only with documented atomicity and human approval. |

### Mandatory routing overrides

- **High-risk security, privacy, authorization, destructive migration, financial correctness or irreversible data changes:** Opus review is the default recommendation regardless of size. Add domain/security/legal/operations specialist review where needed; a model review does not replace it.
- **New architectural decision or unresolved cross-cutting design:** resolve and record the decision in the ADR before implementation. Opus planning/review is recommended; do not delegate an ambiguous design to Haiku.
- **Low confidence:** add discovery/planning before implementation. Do not route directly to a low-cost implementer while a blocking uncertainty remains.
- **Specialist work:** recommend an appropriately skilled human or specialist agent/model. Do not route to Haiku solely to reduce cost.
- **XL:** split before normal implementation. If atomicity is claimed, document why and require explicit human approval.
- **Any model/tool restriction or existing project policy:** follow that policy and record the deviation from this recommendation.

Use exact model names only when the project has selected and made them available. Otherwise use role labels (strong planner, economical implementer, independent reviewer) and note that model mapping is pending.

## 9. Phase 3 completeness gate

Before declaring the backlog ready for execution, verify:

1. Every ticket has all required classification fields, including confidence, evidence, rationale and execution strategy.
2. All five score dimensions are present, each is 1–4, the total is arithmetically correct, and the complexity label follows the score plus guardrails/overrides.
3. Risk is assessed independently from complexity and has a concrete rationale.
4. High-risk tickets have High or explicitly justified verification effort, relevant negative-path tests, and an explicit reviewer/specialist gate.
5. XL tickets are split, or have a documented atomicity reason and human approval requirement.
6. Low-confidence tickets identify missing evidence and a discovery step or blocking decision.
7. Execution recommendations are consistent with complexity, risk, architectural impact, confidence and mandatory overrides.
8. Every requirement and scenario maps to one or more tickets; every ticket maps back to requirements/scenarios or explicitly states none.
9. Dependencies are acyclic and do not point to later phases; each phase has an exit criterion.
10. Tickets remain vertical slices or document the exceptional foundation/atomicity rationale.

If any check fails, correct the backlog or list the exact unresolved blocker. Never silently omit a classification dimension or invent repository evidence to pass the gate.

## 10. Measurement feedback loop

Treat recommended routes as hypotheses. When implementation telemetry is available, compare outcomes by complexity, risk and strategy: total and per-phase tokens/cost, first-pass acceptance, correction passes, verification failures, escaped defects and human review effort. Compare like-for-like ticket profiles and note sample size/confounders. Do not claim a route is cheaper or better from raw averages when ticket mix differs. Use measured results to revise the routing table deliberately; do not let one ticket automatically rewrite policy.
