---
description: Start a ticket on its own branch, do the work, run checks and review, then hand you a human checklist
argument-hint: '<ticket id or title>'
allowed-tools: Read, Write, Edit, Glob, Grep, Agent, Skill, AskUserQuestion, Bash
---

Run one ticket end to end up to the human review. Follow the user's own delegation rules (their
CLAUDE.md or settings) for who writes which code. Never push.

0. **Handoff.** If `<ticketsDir>/.handoff.md` exists, read it, then delete it (it is gitignored, so this
   is safe) so a stale handoff never loads twice. Trust the repo files over its text.
1. **Config and tree.** Read `.ticketflow.json`. Run `git status --porcelain --untracked-files=no`; if
   any tracked file is modified or staged, STOP and tell the user to commit or stash their own work
   first. Never stash, reset or discard it yourself. Untracked files do not block: mention them once
   and leave them alone, and record their list (`git ls-files --others --exclude-standard`) in the review
   file under "Untracked before start" so `/ticketflow:done` never commits them.
2. **Ticket.** Find the ticket for `$ARGUMENTS` by id or title under `<ticketsDir>`. No match and the
   argument is a title: create a one-off ticket `<ticketsDir>/adhoc/A-<NN>-<slug>.md` from
   `${CLAUDE_PLUGIN_ROOT}/templates/ticket.md` (ask for goal and acceptance if unclear). If any
   `Depends on` ticket is not ticked in `PROGRESS.md`, warn and ask before continuing.
3. **Branch.** Name it `<type>/<id lowercase>-<slug>` (type from the ticket). Run
   `git switch -c <name> <baseBranch>`. Set the ticket Status to "In progress <date>".
4. **Work.** Implement only what the ticket's goal and acceptance require. Do not commit; the
   changes stay uncommitted until `/ticketflow:done`.
5. **Checks.** Compute the changed files (`git diff --name-only <baseBranch>` plus untracked files).
   Run every command in `checks`, replacing `{files}`. Record each as pass or fail with the real
   output. Never claim a pass that was not run. Fix real failures, then re-run.
6. **AI review.** If `aiReview` is true, review the full diff for correctness and security bugs
   (use the `code-review` skill when it is installed, otherwise review it yourself). Fix confirmed
   bugs, re-run the checks, and list what remains.
7. **Review file.** Write `<ticketsDir>/reviews/<id>.md` from `${CLAUDE_PLUGIN_ROOT}/templates/review.md`:
   diff stat, check results, review findings, each acceptance item with evidence or "no evidence",
   known gaps, and a **Human checklist**: concrete steps the user should open and try, in order.
8. **Stop.** Print the Human checklist in chat and wait. Do not merge or tick anything.
   - User reports a problem: fix it, re-run the checks, update the review file, show the checklist again.
   - User says "done" or approves: read `${CLAUDE_PLUGIN_ROOT}/commands/done.md` and follow it.
