# ticketflow

A Claude Code plugin with **three commands**. It turns a plan into phases and tickets, runs each
ticket on its own branch with an automatic review, and merges only when you say "done". Then it
hands off to a fresh session for the next ticket.

> **Status: v0.1 built, manifest validated with `claude plugin validate`, NOT yet run end to end.**
> The commands, the vendored-skill override and the post-`/clear` hook are untested in a real
> session. Expect fixes after the first real use.

## The flow

```
 (you plan first: grill, plan mode, or a spec file; the plan lives in the conversation or a file)

 /tickets [spec]     runs the vendored to-tickets skill, with local files as the tracker:
                       1. slices the plan into vertical tracer-bullet tickets with blockers
                       2. shows a numbered list; you iterate until you approve
                       3. groups them into phases and writes tickets/phase-N/*.md
                          + tickets/PROGRESS.md   (creates tickets/ and the config on first use)

 /start P3-15        1. branch feat/p3-15-slug from the base branch
                     2. does the work (your own delegation rules apply)
                     3. runs the checks and an AI review automatically
                     4. writes tickets/reviews/P3-15.md and tells YOU what to check:
                          "Open /members, click Reset password, confirm ..."
                     5. STOPS and waits

 you: "done"         1. squash into ONE commit, merge --no-ff into the base branch
 (or /done)          2. tick the ticket and PROGRESS.md, write notes
                     3. write .handoff.md (gitignored)
                     4. say: "Type /clear"

 you: /clear         the new session opens with .handoff.md already loaded
                     it proposes the next ticket:  /start P3-16
```

If you reply to step 4 of `/start` with a problem instead of "done", it fixes it, re-runs the
checks and shows the checklist again.

`/clear` is built into Claude Code. A plugin cannot run it, so typing it is the one manual step.
The command is `/tickets`, not `/plan`, because `/plan` is a built-in Claude Code command.

## Git picture

```
 development
  o--- baseline
  |
  |        feat/p3-13-agent-chat
  |        o--o--o   WIP commits on the branch
  |       /        \
  o-----o          M1    one squashed commit, merged --no-ff
  |                /|
  o---------------o-+-------------------------
  |                  \
  |                   \   fix/p3-16-login-typo
  |                    o--o
  |                   /    \
  o------------------o      M2
```

## Commands

| Command | What it does |
| --- | --- |
| `/tickets [spec path or issue]` | Runs the `to-tickets` skill on the plan in the conversation (or the given spec). Local-files tracker: one ticket file per slice under `tickets/phase-N/`, `PROGRESS.md` updated. Nothing is written before you approve the list. |
| `/start <id or title>` | Refuses modified or staged tracked files. Creates `<type>/<id>-<slug>`, does the work, runs checks and review, writes the review file, prints the human checklist, stops. A title that matches no ticket creates a one-off ticket first. |
| `/done` (or just say "done") | Squash, `--no-ff` merge, tick, notes, delete the local branch, write the handoff, tell you to `/clear`. |

Branch types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`. They match conventional-commit
types, so branch names and commit messages agree.

## Tickets: the vendored skill

`skills/to-tickets/` is [Matt Pocock's `to-tickets` skill](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md),
copied unchanged under its MIT license (`skills/to-tickets/LICENSE`). Its rules:

- each ticket is a narrow end-to-end slice, demoable alone and small enough for one context window
- each ticket lists the tickets that block it
- you review the numbered list and iterate before anything is published
- tickets describe behaviour and acceptance criteria, not file paths or code
- a wide mechanical refactor is sequenced as expand, migrate in batches, contract

The skill publishes to a tracker configured by another of his skills, which we do not use. So the
`/tickets` command tells it, as an override: **the tracker is local files**. Instead of
`.scratch/<slug>/issues/`, write to `tickets/phase-N/<id>-<slug>.md` with our template, map
"Blocked by" to `Depends on`, and add a `PROGRESS.md` line per ticket. The override lives in
`commands/tickets.md`; the skill file itself is never edited, so it can be updated from upstream.

Grilling is a separate step you run before `/tickets` (your own `grilling` skill, or plan mode).

## Review file

Written by `/start` at `tickets/reviews/<id>.md`:

- diff stat against the base branch
- each configured check: pass or fail, never assumed
- AI review findings
- each acceptance item with its evidence, or "no evidence"
- **Human checklist**: concrete things for you to open and try, in order
- known gaps and risks

`/done` refuses when the file is missing, and refuses failed checks unless you accept them in
writing.

## Handoff

`/clear` wipes the chat, so the summary lives in a file: `tickets/.handoff.md`. It is listed in
`.gitignore`, overwritten by each `/done`, and never committed. A start-up hook loads it into the
new session automatically, after `/clear` and on a fresh `claude` start. `/start` deletes it once
read, so it never loads twice.

It is built from the repo files, not from the chat:

- what shipped, and the commit id
- state: branch, tree clean or not, deploy or container status
- open items and known gaps carried over
- the next ticket (first unticked with all blockers done) and why
- gotchas learned: commands, platform quirks, rules
- things you must do, by name only. It never contains keys, tokens, passwords or invite codes.

The new session trusts the repo files over the handoff text, and tells you if the branch or tree
does not match.

## Config: `.ticketflow.json`

```json
{
  "baseBranch": "development",
  "ticketsDir": "tickets",
  "idFormat": "P{phase}-{nn}",
  "autoCommitDocs": true,
  "aiReview": true,
  "checks": [
    "node_modules/.bin/oxfmt --check {files}",
    "node_modules/.bin/oxlint {files}",
    "tsc --noEmit",
    "vitest run"
  ]
}
```

- `{files}` expands to the files changed on the branch.
- `autoCommitDocs`: `/tickets` and one-off tickets commit their docs onto the base branch so
  `/start` always finds a clean tree. Set `false` to commit them yourself.
- The plugin assumes nothing about your toolchain. Checks come only from this file.

## Repo layout it manages

```
tickets/
  PROGRESS.md                one line per ticket, grouped by phase, with gates
  .handoff.md                 gitignored, overwritten by /done
  phase-1/P1-01-slug.md      ticket files
  notes/P1-01-findings.md    evidence written at merge
  reviews/P1-01.md           review file written by /start
.ticketflow.json
```

Ticket template (the skill's local template, extended with our fields):

```
# P3-15: Title

| Field | Value |
| --- | --- |
| Depends on | ticket ids, or "None (can start immediately)" |
| Status | [ ] Planned <date> |

**What to build:** the end-to-end behaviour, from the user's perspective.

## Acceptance
- [ ] criterion 1
- [ ] criterion 2

## Notes
```

## Rules baked in

- No command pushes. Pushing is always yours.
- `/start` refuses modified or staged tracked files (untracked files only get a warning), so work never leaks between tickets.
- Merging and ticking happen only on your explicit "done".
- Never inferred from silence or from an agent's report.
- No secret value is written to a ticket, review, note or handoff.
- Existing user changes are never overwritten, stashed or reset by a command.

## Install (planned)

Create a GitHub repo for this folder and push it yourself, then:

```
/plugin marketplace add <owner>/ticketflow
/plugin install ticketflow@ticketflow
```

## Plugin layout (planned)

```
ticketflow/
  .claude-plugin/plugin.json
  .claude-plugin/marketplace.json
  commands/tickets.md  start.md  done.md
  skills/to-tickets/SKILL.md  LICENSE       vendored, unchanged, MIT, Matt Pocock
  hooks/hooks.json                          start-up hook that loads tickets/.handoff.md
  templates/ticket.md  review.md  handoff.md  progress.md  ticketflow.json
  README.md
```

Commands are markdown prompts. The hook is one line that prints the handoff file. There are no
other scripts.

## Credits

`to-tickets` is by Matt Pocock, MIT licensed: https://github.com/mattpocock/skills

## Out of scope

- Pushing, GitHub PRs, CI. The review file is the local stand-in for a PR.
- Delegation rules (Codex, other agents). Those stay in your own settings.
- Clearing the session. Claude Code owns `/clear`.

## Open items (untested)

- Confirm a command can read the vendored skill by `${CLAUDE_PLUGIN_ROOT}` path (the skill has
  `disable-model-invocation: true`). If not, inline its steps into `commands/tickets.md`.
- Confirm the `SessionStart` hook with matcher `clear` injects `tickets/.handoff.md` after `/clear`.
  If not, add a tiny `/resume` command.
- Confirm the `autoCommitDocs` default.
- The plugin has no license of its own yet; choose one before publishing. The vendored skill keeps
  its own MIT license.
