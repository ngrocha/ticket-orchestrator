# ticket-tools — a Claude Code plugin marketplace

One plugin: **ticket-orchestrator**. It turns a Markdown backlog ticket into
reviewed, verified code in a single Claude Code invocation.

The main session (run it on Opus) does the thinking: it finds the ticket, explores
only what the ticket needs, makes the design decisions, writes a bounded
implementation brief, and acts as the acceptance authority. A Haiku subagent does
the typing inside the scope it was given. The orchestrator then reads the real diff
— not the worker's summary — and either accepts it or sends back a narrow list of
blocking corrections.

Nothing is copied into your application repository. There is no runtime, no service,
no database. The whole plugin is four Markdown and JSON files.

---

## Installation

```
/plugin marketplace add OWNER/REPOSITORY
/plugin install ticket-orchestrator@ticket-tools
```

`ticket-tools` is the marketplace name from `.claude-plugin/marketplace.json`, not
the repository name. The install command opens the plugin details panel so you can
review what it adds and choose a scope (`user`, `project` or `local`) before
confirming.

From your shell instead of a session:

```bash
claude plugin marketplace add OWNER/REPOSITORY
claude plugin install ticket-orchestrator@ticket-tools
```

Shell installs load on the next Claude Code start, or after `/reload-plugins` in an
open session.

---

## Usage

Start Claude Code on Opus in your application repository, then:

```
/ticket-orchestrator:run ABC-123
```

Other forms:

```
/ticket-orchestrator:run                              # pick the first ready ticket and say which
/ticket-orchestrator:run ABC-123 --backlog docs/backlog.md
/ticket-orchestrator:run ABC-123 --auto-commit        # commit without asking first
```

One ticket per invocation. Run it again for the next one.

### Backlog discovery

With no `--backlog` flag, the skill looks for, in order:

1. `BACKLOG.md`
2. `docs/backlog.md`
3. `backlog.md`

If none exists it reports those paths and stops. It will not invent a backlog.

### What happens

| Phase | Who | What |
|---|---|---|
| 1 | Opus | Find the backlog and ticket, check status, dependencies and working tree |
| 2 | Opus | Explore only the relevant code; find the repo's real verification commands |
| 3 | Opus | Write a bounded implementation brief |
| 4 | Haiku | Implement inside the brief's scope, add tests, run focused checks |
| 5 | Opus | Read the actual diff and tests; run verification; accept or reject |
| 6 | Haiku | Fix only the blocking issues — up to three worker passes in total |
| 7 | Opus | Report results, suggest a commit message, ask before committing |

The orchestrator never creates a branch or a worktree, never pushes, never merges,
and never commits to a default branch without explicit confirmation.

---

## Recommended backlog format

There is no parser and no required frontmatter. Ordinary Markdown works. This format
reads cleanly for both humans and the orchestrator:

```markdown
## ABC-123 — Add health endpoint

**Status:** Ready
**Depends on:** None

### Description

Add an HTTP health endpoint for deployment probes.

### Acceptance criteria

- `GET /health` returns HTTP 200.
- The response follows the existing API response conventions.
- Automated tests cover successful access.

### Notes

Kubernetes readiness probes will call this every 10 seconds.
```

Useful in practice:

- **A stable ID per ticket.** Everything else is optional; this is what you pass on
  the command line.
- **A status value** that reads as ready or not (`Ready`, `To Do`, `In Progress`,
  `Done`, or a `- [ ]` checkbox).
- **Acceptance criteria as a list.** This is what the orchestrator reviews against,
  so vague criteria produce vague review. If the criteria are unusable it will stop
  and ask rather than guess.
- **Dependencies by ID**, when a ticket genuinely cannot start first.

Headings, bullet lists, tables and checkboxes are all tolerated. Keep one ticket per
heading so the boundaries are unambiguous.

---

## Repository layout

```
REPOSITORY/
├── .claude-plugin/
│   └── marketplace.json              # the marketplace catalog
├── plugins/
│   └── ticket-orchestrator/
│       ├── .claude-plugin/
│       │   └── plugin.json           # the plugin manifest
│       ├── agents/
│       │   └── ticket-implementer.md # the Haiku worker
│       ├── skills/
│       │   └── run/
│       │       └── SKILL.md          # the orchestrator, /ticket-orchestrator:run
│       └── README.md
├── LICENSE
└── README.md
```

`skills/run/` is what makes the command `/ticket-orchestrator:run` — the skill's
directory name becomes the command name, namespaced by the plugin name.

---

## Local testing before publishing

**1. Validate the plugin and the marketplace.**

```bash
claude plugin validate ./plugins/ticket-orchestrator
claude plugin validate .
```

Both should end with `✔ Validation passed`.

**2. Load the plugin directly, without a marketplace.**

From an application repository that has a backlog:

```bash
claude --plugin-dir /path/to/REPOSITORY/plugins/ticket-orchestrator
```

Then check that `/ticket-orchestrator:run` appears in the `/` menu, and that
`ticket-implementer` shows up in `/agents`.

**3. Install from the local marketplace directory.** This is the same path your
users take, so it catches marketplace-level mistakes that step 2 misses:

```
/plugin marketplace add /path/to/REPOSITORY
/plugin install ticket-orchestrator@ticket-tools
```

**4. Run it against a throwaway ticket.** Create a trivial ticket in a scratch repo
— add a constant, add one test — and run the full loop. Watch for:

- It names the backlog file and the ticket it chose.
- It reports the working tree state and does not touch your uncommitted work.
- It actually invokes `ticket-implementer` (the subagent appears in the task list).
- It reads the diff itself after the worker returns, rather than echoing the summary.
- It runs your repository's real commands, not invented ones.
- It asks before committing.

**5. Test the failure paths.** These are the ones that matter:

- Run it in a repository with no backlog → it should list the three paths and stop.
- Pass an ID that does not exist → it should list the IDs it found and stop.
- Write a ticket with a deliberately impossible criterion → it should exhaust the
  three worker passes and report honestly instead of declaring success.

**6. Iterate.** Edits to the plugin files take effect on the next session, or run
`/reload-plugins`. If you installed from a local marketplace path, run
`/plugin marketplace update ticket-tools` after changing `marketplace.json`.

**7. Publish.** Push to GitHub and have someone else run the two install commands at
the top of this README from a clean machine.

---

## Validation checklist

Before you tag a release:

- [ ] `claude plugin validate ./plugins/ticket-orchestrator` passes.
- [ ] `claude plugin validate .` passes.
- [ ] The entry `name` in `marketplace.json` matches the `name` in `plugin.json`
      (`ticket-orchestrator`). A mismatch installs the plugin under a name that
      breaks its namespaced command.
- [ ] The entry `source` is a path relative to the marketplace root — the directory
      containing `.claude-plugin/`, not `.claude-plugin/` itself.
- [ ] `OWNER`, `REPOSITORY`, `AUTHOR_NAME` and `AUTHOR_EMAIL` are replaced
      everywhere, including in this README.
- [ ] `/plugin` → Errors tab is empty after install.
- [ ] `claude plugin details ticket-orchestrator` lists one skill and one agent.
- [ ] A real ticket runs end to end on Opus.

---

## v1 scope, and what is deliberately left out

**In v1:**

- One orchestrator skill, one Haiku worker, one ticket per invocation.
- Backlog discovery with an override flag.
- Bounded correction loop, three worker passes.
- Diff-based review by the orchestrator, with real verification commands.
- Opt-in commit; no branching, no pushing, no merging.

**Deliberately not in v1:**

| Not built | Why |
|---|---|
| Parallel tickets / worktrees | Needs isolation, merge handling and failure recovery. That is a different product, and it hides review quality problems behind throughput. |
| A backlog parser or required frontmatter | Backlogs are human documents. A schema would be the first thing users fight. |
| A separate explorer or reviewer subagent | Exploration findings and review judgment are exactly what the orchestrator needs in its own context. Delegating them costs the quality the design is built around. |
| Hooks | Nothing here needs to fire on an event. The skill and agent instructions cover the whole lifecycle. |
| Persisted run state | A run is one conversation. Anything worth keeping is in git or in the backlog. |
| An MCP server or any runtime | There is no external system to reach. Markdown and JSON are sufficient. |

**Plausible later, roughly in order:** a `--dry-run` that stops after the brief so
you can inspect it; a configurable pass budget; a Sonnet worker option for tickets
with real logic; an opt-in worktree flag; multi-ticket runs once single-ticket runs
are boring.

---

## License

MIT. See [LICENSE](LICENSE).
