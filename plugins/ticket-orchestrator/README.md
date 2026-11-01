# ticket-orchestrator

Run one backlog ticket end to end. The main Opus session locates the ticket,
explores only what it needs, writes a bounded implementation brief, delegates the
coding to a Haiku subagent, then reviews the actual diff and runs the repository's
verification commands before accepting anything.

## Install

```
/plugin marketplace add OWNER/REPOSITORY
/plugin install ticket-orchestrator@ticket-tools
```

## Use

Start Claude Code on Opus in your application repository:

```
/ticket-orchestrator:run ABC-123
/ticket-orchestrator:run ABC-123 --backlog docs/backlog.md
/ticket-orchestrator:run --auto-commit
```

With no ticket ID it picks the first ready ticket and tells you which one. Backlog
discovery order: `BACKLOG.md`, `docs/backlog.md`, `backlog.md`.

## Contents

- `skills/run/SKILL.md` — the orchestrator, invoked as `/ticket-orchestrator:run`
- `agents/ticket-implementer.md` — the Haiku implementation worker

Full documentation, backlog format and testing instructions:
https://github.com/OWNER/REPOSITORY
