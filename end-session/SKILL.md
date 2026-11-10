---
name: end-session
description: Close a work session properly. Gathers the facts from git, asks you for notes for next time, writes a standalone session summary into the project's Session-Summary/ folder, commits that one file (and nothing else), and tells you when everything is saved and safe to close, so /start-session can pick it up next time. Use when the user runs /end-session, says "end the session", "wrap up", or is finishing work on a project and wants a record kept.
user-invocable: true
allowed-tools: "Read Glob Grep Bash Write AskUserQuestion"
---

# End Session

Close a session so the next one does not start from nothing.

This writes one summary per session to `<project-root>/Session-Summary/`,
commits it, and confirms that everything is saved. `/start-session` reads it
back next time.

## Step 1. Gather the facts. Run commands, do not recall.

Project root: the git root if there is one (`git rev-parse --show-toplevel`),
otherwise the current working directory.

```
git -C <project-root> log --oneline <start-commit>..HEAD
git -C <project-root> status --short
git -C <project-root> branch --show-current
git -C <project-root> rev-parse --short HEAD
```

`<start-commit>` is, in order of preference:

1. the commit `/start-session` reported at the start of this session;
2. the "Closed at commit" line of the newest summary in `Session-Summary/`;
3. if neither exists, use `git log --oneline --since="<today> 00:00"`
   instead.

Say which one you used.

If the project has a test command, run it and keep the real result. If it has
none, the summary says so. Never write down a test number you did not see in
this session.

Also read the newest file in `Session-Summary/`, if there is one. Step 3 needs
it.

## Step 2. Get closing notes without ending your turn

Steps 2 to 6 all happen in the same turn as the `/end-session` command. This
skill's tool permissions last only for that turn. If you end your turn to wait
for an answer, saving the file and committing it will each stop and ask the
user to approve.

Get the notes the first way that applies:

1. **The user typed notes after the command** (for example `/end-session add
   the cake recipe next time`): those are the notes. Do not ask again.
2. **The user already answered earlier in this closing exchange:** use that
   answer.
3. **Otherwise, call the AskUserQuestion tool.** It waits for the answer
   inside this turn.
   - question: "Anything you want noted for next session? Something to pick
     up, an idea, a loose thought?"
   - header: "Notes"
   - options: "Nothing to note" first, then up to two short suggestions taken
     from what is still open. The user can also type their own answer.
4. **AskUserQuestion is not available:** ask the same question in plain text
   and wait. Saving may then ask the user for permission, which is expected.

The notes go into the summary close to verbatim, in their own section, because
they are the user's direction for next time, not a fact about this session.
Paraphrasing them risks losing the part that mattered. If the user picked a
suggestion, that suggestion is their note. "Nothing to note" means write
"None", never an invented note.

## Step 3. Write the summary

Write it from the facts gathered in Step 1 and from what actually happened in
this conversation. Facts only: every commit id, file name and test number comes
from output seen in this session. No adjectives about how the session went.

**Make it stand alone, not a delta.** If a previous summary exists, carry
forward every item it listed under Still open, and mark each one done, still
open or dropped, based on what actually happened this session. Someone reading
only the newest file should never need an older one to know what is
outstanding.

Use this layout:

```markdown
# Session summary: <project>, <YYYY-MM-DD>

- **Project:** <project> (`<project-root>`)
- **Date:** <YYYY-MM-DD>
- **Agent:** <agent and model, e.g. Claude Code, claude-sonnet-...>
- **Branch:** <branch>
- **Started at commit:** <hash, or "unknown">
- **Closed at commit:** <HEAD before this summary's own commit>

## Task plan
What this session set out to do, each item marked done, partial or not started.

## Findings
What was learned or decided, especially anything that contradicted what was
believed at the start. Include the evidence, not only the conclusion.

## Progress
What was actually done, each item with its commit id. Items with no commit say
"uncommitted". Real test results go here.

## Still open
Each item, with what blocks it.

## Unverified
Anything claimed or built but not checked this session. Always present. "None"
is a valid value.

## Notes for next session
The user's words from Step 2, close to verbatim, or "None".
```

## Step 4. Save the file

Path: `<project-root>/Session-Summary/session-<project>-<YYYY-MM-DD>.md`, where
`<project>` is the project folder's name with spaces replaced by hyphens.

- Create `Session-Summary/` if it does not exist.
- Never overwrite an existing summary. A second session on the same day gets
  `-2`, a third `-3`, and so on.
- Write the file before touching git. It is the record, and it must not depend
  on any git step succeeding.

## Step 5. Commit the summary, and only the summary

Skip the commit, and say in one line why, when any of these holds:

- The project is not a git repo (`git -C <project-root> rev-parse
  --is-inside-work-tree` fails). Do not `git init` one.
- HEAD is detached (`git -C <project-root> symbolic-ref -q HEAD` exits
  non-zero). A commit there belongs to no branch and is easy to lose.
- `git -C <project-root> status` reports a merge, rebase, cherry-pick or
  revert in progress. Git refuses this kind of commit during a merge, but it
  would quietly slip one into a paused rebase.

Otherwise, commit on whatever branch is checked out:

```
git -C <project-root> add -- Session-Summary/<filename>
git -C <project-root> commit -m "Session summary: <project> <YYYY-MM-DD>" -- Session-Summary/<filename>
```

The trailing `-- <path>` is what keeps this safe. Git commits only that file,
even when the user has other changes staged, and leaves everything else exactly
as it was. So never use `git add -A`, `git add .` or `git commit -a` here,
because they would sweep the user's unfinished work into a commit labelled as a
session summary.

- If `git add` refuses because `Session-Summary/` is gitignored, the user chose
  that. Do not force it with `-f`. Report it and move on.
- If a pre-commit hook fails, never retry with `--no-verify`. Report the hook's
  output.
- Never push. A local commit is easy to undo, a push publishes.

**Then check the result, not the exit code:**

```
git -C <project-root> show --name-only --format=%h HEAD
git -C <project-root> status --short -- Session-Summary/<filename>
```

The first must list exactly one file, the summary. The second must print
nothing. The summary cannot contain its own commit id, so the id goes in the
closing message, not into the file. Do not amend the commit to add it.

## Step 6. Tell the user it is safe to close

Close with one short message. When the file is on disk and both checks in
Step 5 passed:

> All saved. Summary written to `Session-Summary/<filename>` and committed as
> `<hash>`. You can close this session. Next time, run `/start-session` in this
> project and it will pick up from here.

If the commit was skipped or failed, say "saved but not committed", with the
reason. Closing is still safe, because the file is on disk.

If `git status` in Step 1 showed other uncommitted changes, list them after
that message: "These files still have uncommitted changes. The summary did not
commit them: ...". Closing the session does not lose them, but the user should
know they are there.

Only say "All saved" after you have seen the file on disk and the commit checks
pass.

## Rules

- Never invent a fact. Every number comes from a command run this session.
- Commit the summary, only the summary, and never push.
- Never overwrite an existing summary.
- The local file is never optional. If git fails, the file still gets written.
