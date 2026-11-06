---
name: run
description: Run exactly one backlog ticket end to end. Plans with the main-session model, delegates bounded implementation to the ticket-implementer Haiku subagent, independently reviews the real diff, runs verification, performs at most two correction passes, and emits a structured telemetry record.
argument-hint: "[ticket-id] [--backlog <path>] [--auto-commit]"
disable-model-invocation: true
allowed-tools: Agent, Read, Grep, Glob, Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git branch *), Bash(ls *)
---

# Ticket orchestration

Arguments: `$ARGUMENTS`

You are the orchestrator. You run on the main session's model. Use Opus for this workflow when the user wants the Opus → Haiku architecture. You own ticket selection, exploration, architecture, design, risk, the implementation brief, independent review, acceptance and telemetry. The `ticket-implementer` subagent runs on Haiku and owns only the bounded code changes described in your brief.

**One invocation handles exactly one ticket.** Follow the phases in order. Do not skip ahead, run multiple tickets, or let the worker select work.

The `allowed-tools` frontmatter pre-approves only the listed tools/commands for this skill invocation; it is not a security sandbox or an exhaustive tool allowlist. Commands not pre-approved remain subject to the user's normal Claude Code permission rules. Never broaden permissions to avoid a prompt.

## Parsing the arguments

- First bare token, if present, is the ticket ID.
- `--backlog <path>` overrides backlog discovery.
- `--auto-commit` means the user has pre-authorised a commit after successful acceptance, subject to the protected/default-branch safeguard in Phase 7.
- No arguments is valid: discover the backlog and select the first ready ticket.
- Unknown options or a missing value after `--backlog` are errors; ask the user to correct the invocation rather than guessing.

---

## Phase 1 — Locate and validate

### Find the backlog

Use `--backlog <path>` if given. Otherwise try, in order:

1. `BACKLOG.md`
2. `docs/backlog.md`
3. `backlog.md`

If none exists, stop cleanly. Tell the user those three paths were checked and that they can pass `--backlog <path>`. Do not create a backlog or guess work from the codebase.

### Find and read the ticket

- Read the backlog and locate the requested ID. Match tolerantly: the ID may appear in a heading, bullet, table row or bold label.
- If the ID is not found, list the IDs you did find and stop.
- If no ID was supplied, select the first ticket whose status is ready (`Ready`, `To Do`, `Open`, an unchecked box, or no status) and whose dependencies are satisfied. State which ticket you selected and why before continuing.
- Extract the title, status, dependencies, description, acceptance criteria and notes. Missing fields are normal; do not invent them.

### Check the gates

Stop and ask the user if:

- The ticket is already `Done`, `In Review` or equivalent.
- A declared dependency is unsatisfied and the repository does not demonstrate that the dependency work already exists.
- Acceptance criteria are too vague to review objectively. Ask for clarification instead of inventing criteria.
- The ticket requires a decision that cannot safely be made from repository conventions and explicit requirements.

### Check repository state

Run `git status --short --untracked-files=all` and `git branch --show-current`. Also inspect the recent commit style with `git log --oneline -10` when a completion report may need a commit-message suggestion.

- Record the initial working-tree state. Do not attribute pre-existing changes to this ticket.
- Uncommitted changes are not automatically a blocker. Report them, and ask before proceeding if any overlap the ticket's likely scope.
- If the current branch is `main`, `master`, or another obvious default/protected branch, stop and ask whether to continue there or have the user switch first. Do not create branches or worktrees.
- Never discard, stash, overwrite, reset, clean or otherwise rewrite pre-existing work.

---

## Phase 2 — Focused exploration

Build only the understanding this ticket needs.

- Locate relevant modules, routes, handlers, models or components using `Grep` and `Glob`; read only the files needed.
- Read repository instructions when present: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, and relevant local guidance.
- Find real verification commands in `package.json`, `Makefile`, `pyproject.toml`, `justfile`, CI workflows or equivalent. Record exact commands for tests, lint, type-check and build where available. Do not invent commands.
- Read one or two close neighbouring implementations/tests to identify conventions for naming, errors, response shapes and test layout.
- Identify architectural consequences and risks: data-shape changes, public APIs, migrations, auth boundaries and backwards compatibility.
- Determine the narrowest practical write scope. Include every file the worker may need to create or edit in the brief. Distinguish files it may edit from read-only context files.

Stop exploring once you can name the files that will change and the patterns to follow. Do not read unrelated ADRs, tickets or broad product documentation.

---

## Phase 3 — Implementation brief

Write a bounded, self-contained brief. The worker receives this brief and the explicit worker constraints only; assume it does not see the backlog, this conversation or your exploration notes.

Include exactly these sections:

```markdown
# Ticket <ID> — <objective in one line>

## Acceptance criteria
<only this ticket's criteria, verbatim where possible>

## Required behaviour
<concrete required behaviour>

## Non-goals
<what must not change; include traps discovered during exploration>

## Files in scope
<explicitly label each path EDIT or READ-ONLY, with one line explaining why>

## Patterns to follow
<named existing examples, e.g. mirror the error shape in src/api/users.ts>

## Constraints and invariants
<signatures, data, compatibility, security, dependency and scope constraints>

## Expected tests
<behavioural tests to add/update and where>

## Verification commands
<exact commands copied from the repository>

## Definition of done
<a short checklist the worker can self-assess against>
```

Brief rules:

- Concrete beats general. Specify observable outcomes and edge cases.
- Do not speculate or include background the worker cannot act on.
- Do not paste full files unless a short excerpt is necessary to demonstrate a pattern.
- Make design decisions here. The worker must not choose architecture.
- The `Files in scope` section is an edit allowlist. Files marked READ-ONLY may be inspected but not changed.
- If a new file is a legitimate implementation requirement, list its exact path as EDIT.
- Include exact test/lint/type-check/build commands only when discovered. If a command does not exist, say `Not present in repository` rather than making one up.

---

## Phase 4 — Delegate implementation

Invoke the `ticket-implementer` subagent through the `Agent` tool, using the subagent type `ticket-implementer`. Pass the complete brief.

The handover must explicitly state:

- This is the initial implementation pass (pass 1 of a maximum 3).
- Only paths marked EDIT may be changed. READ-ONLY paths are context only.
- Do not change the backlog or ticket status.
- Do not commit, stage, push, merge, switch branches, create branches/worktrees, stash, reset or clean.
- Do not install or modify dependencies unless the brief explicitly authorises the exact change.
- Do not expand scope, make unresolved architecture decisions or spawn agents.
- Run only the verification commands in the brief, plus narrowly scoped read-only inspection needed to understand a failure.
- Return the structured worker result defined by the `ticket-implementer` agent.

Wait for the worker. Treat its summary and claims as unverified until independently checked.

---

## Phase 5 — Review the actual changes

The worker's summary is not evidence. Review repository state independently.

1. Run `git status --short --untracked-files=all`, `git diff --stat`, `git diff --`, and `git diff --cached` (if there are staged changes).
2. Compare the current state with the Phase 1 baseline. Separate pre-existing changes from this ticket's changes.
3. Inspect **every** changed tracked file and **every** untracked file attributable to this ticket. Ordinary `git diff` does not show untracked files; open/read each one directly. Do not overlook newly created source files, tests, fixtures or generated files.
4. Confirm every edit is within the brief's EDIT allowlist. Any out-of-scope edit is blocking unless the user explicitly approves the scope change.
5. Read the tests themselves. Do not infer test quality from filenames or the worker's summary.

Review all of the following:

1. Ticket requirements — is the requested behaviour implemented?
2. Acceptance criteria — assess each criterion individually.
3. Architecture/design intent — does the change match the brief?
4. Repository conventions — does it match neighbouring code?
5. Correctness — edge cases, null/empty values, concurrency, off-by-one and error paths.
6. Test quality — tests assert observable behaviour and would fail if the behaviour regressed. Weakening, skipping or deleting an existing test is blocking.
7. Error handling — no swallowed failures or misleading success paths.
8. Security — injection, authorization, secrets, unvalidated input, unsafe deserialization and dependency changes.
9. Scope discipline — no unrequested edits, refactors or generated noise.
10. Maintainability — the next engineer can understand the change.

Then run the exact verification commands from the brief yourself, narrowed to the affected area only where the repository supports that and without changing the meaning of the check. Do not invent substitute commands. If a command needs permission, request it through normal Claude Code permission flow. If it cannot run, report it as NOT RUN and why. Never infer success.

Reach a verdict: **ACCEPT** or **REJECT**. Acceptance requires every blocking criterion to pass and all required verification to pass, or an explicit, clearly documented reason a check is unavailable that the user accepts. Do not silently waive a failed check.

---

## Phase 6 — Bounded correction loop

Budget: **three worker passes total** — initial implementation plus at most two correction passes. Announce the pass count in the user-facing progress/report.

On rejection, produce:

```markdown
## Blocking
1. <file:line> — what is wrong, why it blocks, and the exact correction required

## Non-blocking
- <future improvement; do not send to the worker>
```

Delegate only the blocking items. The correction handover must include:

- Original ticket ID and acceptance criteria.
- The specific blocking issues, with file/line references where available.
- Exact files the worker may edit for this correction.
- Relevant verification commands.
- Current pass number and remaining pass budget.

Do not resend broad exploration notes or the full repository analysis. Do not let the worker address non-blocking suggestions.

After each correction, repeat Phase 5 against the actual updated diff and rerun relevant verification. Also check that the correction did not regress previously accepted criteria.

When the budget is exhausted, stop and report honestly:

- Acceptance criteria still unmet.
- What was attempted in each pass.
- Relevant failing command output.
- Whether changes are safe to keep, safe only behind a flag, or should be reverted.
- If recommending a revert, provide a precise command scoped only to this ticket's changes. Never execute it.
- What a human should inspect first.

Never present failing verification as success or soften a blocker into a minor concern.

---

## Phase 7 — Completion, commit gate and telemetry

### Completion report

On acceptance, report:

- **Summary** — two or three sentences.
- **Files changed** — each path and purpose; separate pre-existing changes.
- **Verification** — exact command and outcome for tests, lint, type-check and build. List every check skipped and why.
- **Acceptance criteria** — each criterion marked MET / NOT MET with concise evidence.
- **Worker passes used** — e.g. 2 of 3.
- **Commit message** — suggested in the repository's existing style, informed by `git log --oneline -10`.

If rejected or incomplete, use the honest failure report from Phase 6 instead; do not report acceptance.

### Commit gate

- Without `--auto-commit`, ask before staging or committing.
- With `--auto-commit`, stage only the exact ticket files that were independently reviewed and accepted. Never stage unrelated or pre-existing changes. Inspect the staged diff before committing.
- `--auto-commit` does not override the protected/default-branch safeguard. Do not commit on a protected/default branch without explicit confirmation in this turn.
- Never push or merge.
- Do not update the backlog or ticket status automatically. Propose the exact status edit and apply it only after the user agrees.

### Structured telemetry contract

Emit one valid JSON object in the final report under `## Ticket telemetry (JSON)`. This is a measurement record, not a narrative estimate. Use JSON `null` for unknown/unavailable values. Do not fabricate token counts, costs, model IDs, timestamps, pass counts or quality outcomes.

Use this schema (add fields only compatibly; do not rename existing keys):

```json
{
  "schema_version": "1.0",
  "ticket_id": "PH-000",
  "session": {
    "session_id": null,
    "started_at": null,
    "completed_at": null,
    "branch": null
  },
  "models": {
    "orchestrator": {
      "configured": null,
      "resolved": null
    },
    "implementer": {
      "configured": "haiku",
      "resolved": null
    }
  },
  "worker": {
    "passes_used": null,
    "correction_passes": null,
    "rework_required": null
  },
  "outcome": {
    "status": "accepted",
    "acceptance_criteria_total": null,
    "acceptance_criteria_met": null,
    "blocking_issues_remaining": null,
    "scope_violations": null,
    "tests_added_or_changed": null,
    "verification": {
      "tests": "passed",
      "lint": "not_run",
      "typecheck": "not_run",
      "build": "not_run"
    }
  },
  "usage": {
    "attribution_level": "per_phase",
    "currency": "USD",
    "phases": {
      "planning": {
        "model": null,
        "input_tokens": null,
        "output_tokens": null,
        "cache_read_input_tokens": null,
        "cache_creation_input_tokens": null,
        "cost": null,
        "source": "unavailable"
      },
      "implementation": {
        "model": "haiku",
        "input_tokens": null,
        "output_tokens": null,
        "cache_read_input_tokens": null,
        "cache_creation_input_tokens": null,
        "cost": null,
        "source": "unavailable"
      },
      "correction": {
        "model": "haiku",
        "input_tokens": null,
        "output_tokens": null,
        "cache_read_input_tokens": null,
        "cache_creation_input_tokens": null,
        "cost": null,
        "source": "unavailable"
      },
      "review": {
        "model": null,
        "input_tokens": null,
        "output_tokens": null,
        "cache_read_input_tokens": null,
        "cache_creation_input_tokens": null,
        "cost": null,
        "source": "unavailable"
      },
      "verification": {
        "model": null,
        "input_tokens": null,
        "output_tokens": null,
        "cache_read_input_tokens": null,
        "cache_creation_input_tokens": null,
        "cost": null,
        "source": "unavailable"
      }
    },
    "total": {
      "input_tokens": null,
      "output_tokens": null,
      "cache_read_input_tokens": null,
      "cache_creation_input_tokens": null,
      "cost": null,
      "source": "unavailable"
    },
    "notes": []
  }
}
```

Telemetry rules:

- `outcome.status` must be one of `accepted`, `rejected`, `blocked`, `incomplete`.
- Verification values must be `passed`, `failed`, `not_run`, or `not_applicable`.
- `worker.passes_used` counts actual worker invocations, maximum 3. `correction_passes` excludes the initial implementation pass.
- `rework_required` is true if at least one correction pass was needed, false if none was needed, and null if unknown.
- `scope_violations` counts actual worker edits outside the EDIT allowlist; use 0 only after checking tracked and untracked changes.
- `acceptance_criteria_met` counts criteria independently verified as met, not criteria the worker claims to have met.
- For usage, capture only values actually exposed by Claude Code/agent result metadata or an authoritative usage source available in the session. Keep the source name in `source` (for example `agent_result_metadata`, `claude_usage_api`, `claude_usage_session_estimate`, `manual_export`).
- Do not use a session-wide `/usage` total as if it were per-phase or per-agent usage. If only an unattributed session total is available, set `attribution_level` to `session_unattributed`, place that aggregate only in `usage.total`, and explain the limitation in `notes`.
- Claude Code's `/usage` dollar figure may be a local list-price estimate; do not call it authoritative billing. Label it `claude_usage_session_estimate`. If billing-grade cost is unavailable, say so.
- If a value cannot be attributed to a phase/model, leave that phase field null. Never allocate totals proportionally or estimate phase costs.
- `usage.attribution_level` must be `per_phase`, `per_model`, `session_unattributed`, or `unavailable`.
- If telemetry cannot be obtained, still emit the complete JSON schema with null measurements and an explanatory note.
- Do not claim that this skill automatically persists telemetry to a database or external service. The JSON object is the portable record for a later collector/dashboard.

Finish by stating that this invocation handled exactly one ticket and, if discoverable without starting it, name the next ready ticket.
