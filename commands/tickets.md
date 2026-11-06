---
description: Turn the current plan or a spec into phased tickets (vertical slices with blockers) under tickets/
argument-hint: '[spec path or issue number]'
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash(git:*), Bash(ls:*), Bash(mkdir:*)
---

Turn a plan into phased tickets. Write NOTHING until the user approves the list.

1. **Config.** Read `.ticketflow.json` at the repo root. If it is missing, copy
   `${CLAUDE_PLUGIN_ROOT}/templates/ticketflow.json`, set `baseBranch` to the current branch, and ask the
   user (AskUserQuestion) to confirm `baseBranch` and the `checks` commands for this project.
2. **Method.** Read `${CLAUDE_PLUGIN_ROOT}/skills/to-tickets/SKILL.md` and follow its steps 1 to 4
   exactly (gather context from the conversation or `$ARGUMENTS`, explore the code, draft vertical
   tracer-bullet slices with blockers, quiz the user until they approve). Its step 5 is replaced by
   the override below. Never edit that skill file.
3. **Phases.** After the list is approved, group tickets into phases (propose the grouping and let
   the user adjust). IDs follow `idFormat`, for example `P1-01`. A phase gate is one line: what must
   be true when the phase ends.
4. **Override of step 5: the tracker is local files.** For each ticket write
   `<ticketsDir>/phase-<N>/<id>-<slug>.md` from `${CLAUDE_PLUGIN_ROOT}/templates/ticket.md`:
   "Blocked by" becomes `Depends on`, the acceptance criteria stay as checkboxes, branch type is one
   of feat, fix, chore, docs, refactor, test. No file paths or code in tickets. Create or update
   `<ticketsDir>/PROGRESS.md` from `${CLAUDE_PLUGIN_ROOT}/templates/progress.md`: one unticked line per
   ticket under its phase, plus the gate line.
5. **Ignore the handoff.** Make sure `.gitignore` contains `<ticketsDir>/.handoff.md`.
6. **Commit docs.** If `autoCommitDocs` is true, and the only changes are tickets, config and
   `.gitignore`, commit them as `docs: add tickets for <feature>` and end the message with the
   attribution lines your environment requires. If other changes are present, do not commit; say so.
   Never push.
7. Tell the user the first ticket that can start (no open blockers) and the command:
   `/ticketflow:start <id>`.
