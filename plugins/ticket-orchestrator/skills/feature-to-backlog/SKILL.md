---
name: feature-to-backlog
disable-model-invocation: true
description: "turn a software functionality or feature description into a consistent markdown product documentation set: a PRD addendum, architecture ADR addendum, and implementation backlog of vertical-slice tickets with phases, evidence-based classification, complexity scoring, execution strategy recommendations, dependencies, scenario traceability, requirement coverage, open questions, and deferred items. Use when the user asks to define, specify, plan, backlog, architect, or document a new feature for an existing software product, especially when the output will drive Claude Code, Opus, coding agents, or an engineering team."
---
# Feature to PRD, ADR and Backlog

Create implementation-ready product documentation from a feature description. Treat the user's existing product PRD, ADR, AGENTS.md and backlog conventions as authoritative when available.

## Workflow

1. **Understand the feature.** Extract actors, data, configuration, permissions, workflows/states, edge cases, notifications, reporting, integrations, privacy/security implications and future hooks.
2. **Clarify only material ambiguities.** If decisions would materially change permissions, lifecycle, data model, legal/privacy behavior, or backlog slicing, ask concise questions with a proposed default for each. If the user says “default for everything” or the description is already sufficient, proceed without more questions.
3. **Inspect the existing system context when supplied.** Preserve established requirement IDs, architecture choices, role names, authz patterns, deployment model, notification mechanism, audit model, database/runtime constraints, and ticket numbering. Never redesign implemented foundations without explicitly recording a new ADR decision.
4. **Create exactly three Markdown deliverables by default:**
   - `<project>_<feature>_PRD.md`
   - `<project>_<feature>_ADR.md`
   - `<project>_<feature>_Backlog.md`
   Produce Markdown only unless the user requests other formats.
5. **Slice and classify every ticket.** Apply the vertical-slice rules and the complete scoring, risk, confidence, evidence and execution-strategy rubric in `references/slicing-classification.md`. Do not infer repository-specific facts without inspecting the repository.
6. **Run the Phase 3 completeness gate before finishing.** Verify requirement/scenario coverage, acyclic dependencies, vertical slices, complete classification fields, score arithmetic, evidence, confidence, risk-appropriate verification, XL handling, and routing consistency. Fix omissions or explicitly flag unresolved information; do not silently mark the backlog ready.

Read these references before drafting:
- `references/prd-adr-template.md` for PRD and ADR structure.
- `references/backlog-template.md` for the exact backlog structure and ticket format.
- `references/slicing-classification.md` for vertical slicing, phases, classification, scoring and execution strategy rules.

## Analysis rules

### Preserve product conventions
- Use addenda for a new feature in an existing implemented system rather than rewriting the base PRD/ADR.
- Reuse existing role/audience names, permissions, notification outbox, audit log, jobs, UI kit and infrastructure when they exist.
- Separate **product behavior** (PRD) from **implementation decisions** (ADR) and **delivery units** (Backlog).
- Requirement IDs must be stable and domain-specific (for example `ABS-`, `BAL-`, `GOAL-`). Do not use ticket IDs as requirement IDs.
- Scenarios must use stable IDs and concrete actors/data. Make them independently testable.

### Be opinionated where safe
Fill obvious implementation/product gaps using a documented default. Mark legal, regulatory, HR-policy or jurisdiction-specific defaults as requiring owner/HR/legal validation rather than presenting them as law.

### Tickets are vertical slices
A normal ticket must leave something demoable and testable end to end: data → domain → authz → API → minimal UI where applicable. Do not create horizontal tickets such as “build all database tables”, “build all APIs”, or “create all UI” unless the project explicitly uses a foundation phase for shared enabling work that cannot be demonstrated separately.

Foundation/infrastructure tickets are allowed when genuinely cross-cutting, but minimize them and state the capability they unblock. Classify their actual scope; do not mark them small merely because they have no user interface.

## Output rules

- Write concise, implementation-oriented Markdown.
- No em dashes.
- Use tables for compact mappings and overviews, not for source code.
- Include formulas/code blocks only where they remove ambiguity.
- Use explicit defaults and acceptance examples.
- Do not fabricate repository paths if the repo has not been inspected. In ticket `Implementation`, use likely module/route/table names prefixed with “Expected:” or leave repo-specific paths for planning classification.
- If the user supplies an existing repository or docs, inspect them before filling file paths and module counts.
- Keep classification rationale concise but specific to the ticket. Cite the PRD/ADR requirement, scenario, known module, or repository evidence that supports it.
- Treat execution strategy as a recommendation, not an automatic permission grant or an instruction to bypass human approval. Existing model/tool policies and human approval gates take precedence.

## Backlog planning classification

During planning, classify every ticket using `references/slicing-classification.md`. Put the full classification, score breakdown, rationale, confidence, evidence basis and recommended execution strategy directly in every ticket block. Apply overrides for security/privacy, irreversible changes, unresolved architecture, specialist work and XL scope even when the numeric score suggests a smaller ticket.

Do not claim a model route is optimal based on the rubric alone. It is a starting hypothesis to be evaluated against measured cost, quality, rework and verification outcomes.

## Final response

Return links to the three Markdown files and a short summary containing:
- major defaults/decisions introduced;
- number of phases and tickets;
- classification/routing highlights and any low-confidence items;
- any unresolved question that actually blocks implementation.
Do not repeat the documents in chat.
