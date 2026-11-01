---
name: run
description: Run one backlog ticket end to end from the current repository. Locates the ticket, explores only what it needs, writes an implementation brief, delegates to the ticket-implementer Haiku subagent, reviews the actual diff, corrects within a bounded loop, runs verification, and reports.
argument-hint: "[ticket-id] [--backlog <path>] [--auto-commit]"
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(ls:*)
---

# Ticket orchestration

Arguments: `$ARGUMENTS`

You are the orchestrator. You run on the main session's model — launch Claude Code
with Opus before invoking this skill. You own planning, architecture, design,
risk, review and acceptance. The `ticket-implementer` subagent (Haiku) owns typing
code inside a scope you define.

Process **exactly one ticket** per invocation. Work through the phases in order.
Do not skip ahead, and do not run two tickets because the backlog looks ready.

## Parsing the arguments

- First bare token, if present, is the ticket ID.
- `--backlog <path>` overrides backlog discovery.
- `--auto-commit` means the user has pre-authorised a commit in Phase 7. Without
  it, you must ask before committing.
- No arguments at all is valid: discover the backlog and pick the first ready ticket.

---

## Phase 1 — Locate and validate

**Find the backlog.** Use `--backlog <path>` if given. Otherwise try, in order:

1. `BACKLOG.md`
2. `docs/backlog.md`
3. `backlog.md`

If none exists, stop cleanly. Tell the user those three paths were checked and that
they can pass `--backlog <path>`. Do not create a backlog, and do not fall back to
guessing work from the codebase.

**Find the ticket.** Read the backlog and locate the requested ID. Match tolerantly:
the ID may sit in a heading, a bullet, a table row or a bold label. If the ID is not
found, list the IDs you did find and stop.

If no ID was given, pick the first ticket whose status reads as ready (`Ready`,
`To Do`, `Open`, an unchecked box, or no status at all) and whose dependencies are
satisfied. **State clearly which ticket you selected and why** before continuing.

**Read the ticket.** Extract whatever exists: title, status, dependencies,
description, acceptance criteria, notes. Missing fields are normal — work with what
is there.

**Check the gates.** Stop and ask the user if any of these holds:

- The ticket is already `Done`, `In Review` or similar.
- A declared dependency is not satisfied, and you cannot confirm from the repository
  that the dependency work already exists.
- The acceptance criteria are too vague to review against. Ask for clarification
  rather than inventing criteria.

**Check the working tree.** Run `git status --short` and `git branch --show-current`.

- Uncommitted changes are not automatically a blocker, but you must never discard,
  stash or overwrite them without asking. Report what is already modified, and
  confirm with the user before proceeding if the dirty files overlap the ticket's
  likely scope.
- If the current branch is `main`, `master` or another obvious default, say so and
  ask whether to continue there or whether the user wants to switch branch first.
  **Do not create a branch or a worktree yourself.**

---

## Phase 2 — Focused exploration

Build only the understanding this ticket needs.

- Locate the modules, routes, handlers, models or components the ticket touches.
  Use `Grep`/`Glob` to find them; read the few files that matter.
- Read the repository's own instructions if present — `CLAUDE.md`, `AGENTS.md`,
  `CONTRIBUTING.md`, a style guide in the area you are touching.
- Find the real verification commands from `package.json` scripts, `Makefile`,
  `pyproject.toml`, `justfile`, CI workflow, or equivalent. Record the exact
  commands for test, lint, type-check and build. Do not invent commands.
- Read one or two existing files that are close neighbours of the change, to learn
  the conventions the worker must copy: error handling, test layout, naming,
  response shapes.
- Identify the architectural consequences and the risks: data shape changes, public
  API changes, migrations, auth boundaries, backwards compatibility.

Stop exploring once you can name the files that will change and the patterns to
follow. Do not read unrelated ADRs, other tickets, or broad product documentation.

---

## Phase 3 — Implementation brief

Write a bounded, self-contained brief. The worker sees only this — it does not see
the backlog, this conversation, or your exploration output.

Include exactly these sections:

```
# Ticket <ID> — <objective in one line>

## Acceptance criteria
<only the criteria for this ticket, verbatim where possible>

## Required behaviour
<what the code must do, concretely>

## Non-goals
<what must not change; name the traps you saw during exploration>

## Files in scope
<paths likely to change, and paths worth reading for context, each with one line of why>

## Patterns to follow
<named existing examples: "mirror the error shape in src/api/users.ts">

## Constraints and invariants
<signatures that cannot change, data that cannot be lost, limits to respect>

## Expected tests
<what the tests must cover and where they go>

## Verification commands
<the exact commands, copied from the repository>

## Definition of done
<a short checklist the worker can self-assess against>
```

Rules for the brief:

- Concrete over general. "Return 200 with `{status: "ok"}`" beats "return a healthy
  response".
- No speculation, no background that does not change the implementation, no business
  context the worker cannot act on.
- No full file contents unless a short excerpt is genuinely the clearest way to show
  a pattern.
- If writing the brief exposes a design decision, make it now and record it in the
  brief. The worker must never have to choose an architecture.

---

## Phase 4 — Delegate implementation

Invoke the `ticket-implementer` subagent, passing the brief as its prompt. Add any
explicit constraints: files it may not touch, commands it may not run, a tool it
should avoid.

Say explicitly in the handover that the worker must not commit, must not change the
backlog, must not expand scope, and must not spawn agents.

Wait for its summary. Treat that summary as a **claim**, not as evidence.

---

## Phase 5 — Review the actual changes

Inspect the real diff. `git diff` and `git status --short` first, then read the
changed files where the diff alone is not enough. Open the new tests and read what
they actually assert.

Review against all of:

1. **Ticket requirements** — is the thing that was asked for actually built?
2. **Acceptance criteria** — each one, individually, met or not met.
3. **Architecture and design intent** — does it match the decisions in your brief?
4. **Repository conventions** — does it look like the code around it?
5. **Correctness** — edge cases, null/empty, concurrency, off-by-one, error paths.
6. **Test quality** — do the tests fail if the implementation is wrong? Any test
   that was weakened, skipped or deleted is a blocking issue.
7. **Error handling** — failures surfaced properly, no swallowed exceptions.
8. **Security** — injection, authz checks, secrets, unvalidated input, unsafe
   deserialisation, dependency additions.
9. **Scope discipline** — anything changed that the brief did not ask for.
10. **Maintainability** — would the next person understand this?

Then run the verification commands yourself. If a command needs a permission you do
not have, ask the user to run it and paste the output. Never assume a check passed.

Reach a verdict: **accept** or **reject**.

---

## Phase 6 — Correction loop

Budget: **three worker passes in total** — the Phase 4 implementation plus at most
two corrections. Count them out loud so the user can see where you are.

On rejection, produce a rejection report:

```
## Blocking
1. <file:line> — what is wrong, why it blocks, what correct looks like
## Non-blocking
- <improvements the user may want later; not sent to the worker>
```

Delegate **only the blocking items**. The correction handover contains: the original
brief's acceptance criteria, the specific blocking issues, the files involved, and
the verification commands. Do not resend your exploration notes or the full
repository analysis.

Review the updated diff with the same rigour. Repeat until accepted or the budget is
exhausted.

**When the budget is exhausted, stop and report honestly:**

- Which acceptance criteria are still unmet.
- What was attempted across the passes.
- The relevant failing command output.
- Whether the changes are safe to keep, safe to keep behind a flag, or should be
  reverted — and the exact command to revert if that is your recommendation.
- What a human should look at first.

Never present failing verification as success. Never soften a failure into a
"minor remaining item".

---

## Phase 7 — Completion

On acceptance, report:

- **Summary** — what was implemented, in two or three sentences.
- **Files changed** — the list, with a line each.
- **Verification** — test, lint, type-check and build results, each with its command
  and outcome. **List every check you skipped and why.**
- **Acceptance criteria** — each one, with met / not met.
- **Worker passes used** — e.g. 2 of 3.
- **Commit message** — a concise suggestion in the repository's existing style
  (check `git log --oneline -10`).

Then:

- If `--auto-commit` was passed, stage the changed files and commit. Otherwise **ask
  before committing.**
- Never push, never merge, and never commit onto a protected or default branch
  without explicit confirmation in this turn.
- If the ticket has an unambiguous status field, **propose** the exact edit, for
  example `**Status:** Ready` → `**Status:** Done`, and apply it only once the user
  agrees. Do not rewrite any other part of the backlog.

Finish by stating that this invocation handled one ticket, and name the next ready
ticket if there is one — without starting it.
