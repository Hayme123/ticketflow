---
description: Merge the approved ticket, tick it, write the handoff, then tell the user to /clear
argument-hint: '[ticket id]'
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

Run only after the user has said "done" for this ticket. Never push.

1. **Preconditions.** You are on the ticket's branch and `<ticketsDir>/reviews/<id>.md` exists. If a
   check failed in the review file, STOP unless the user has accepted that failure in writing.
2. **Tick on the branch.** Set the ticket Status to done with today's date, tick the acceptance
   boxes that have evidence, tick its line in `PROGRESS.md`. Write `<ticketsDir>/notes/<id>-findings.md`
   from the review file (what was built, evidence, gaps). Mention in notes anything the user still
   must do, by name only.
3. **One commit.** Stage with `git add -u` (tracked changes) plus only the untracked files that are NOT
   listed under "Untracked before start" in the review file. Never `git add -A`: the user's own
   untracked files must stay out. Then commit as `<type>: <ticket title> (<id>)` (conventional
   commit) with the attribution lines your environment requires. If the user already made commits
   on the branch, keep them; do not rewrite history.
4. **Merge.** `git switch <baseBranch>`, then
   `git merge --no-ff <branch> -m "merge: <id> <ticket title>"`. On conflict, STOP and report; never
   force or reset. Then `git branch -d <branch>`.
5. **Handoff.** Write `<ticketsDir>/.handoff.md` from `${CLAUDE_PLUGIN_ROOT}/templates/handoff.md`, built from the
   repo files (not from memory of the chat): what shipped and the merge commit id, state (branch,
   tree clean, deploy or container status if known), open items and gaps, the next ticket (first
   unticked one whose blockers are all ticked) and why, gotchas learned, and what the user must do
   (by name only). Never write keys, tokens, passwords or invite codes. Confirm `.gitignore`
   lists the file.
6. **Tell the user:** "<id> is merged into <baseBranch>. Type /clear. The next session loads the
   handoff and proposes `/ticketflow:start <next id>`."
