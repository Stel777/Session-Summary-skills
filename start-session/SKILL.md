---
name: start-session
description: Open a work session already knowing what happened before. Reads the last session summaries from the project's Session-Summary/ folder, checks them against the current git state, and briefs you on what was finished, what is still open, what was never verified, and the notes you left for yourself. Use when the user runs /start-session, says "start a session", "pick up where we left off", or comes back to a project after a break.
user-invocable: true
allowed-tools: "Read Glob Grep Bash"
---

# Start Session

Pick up where the last session left off instead of starting from nothing.

This pairs with `/end-session`, which writes one summary per session into
`Session-Summary/` at the project root and commits it. This skill reads those
summaries back. It only reads and reports, it never changes a file.

## Step 1. Find the project root

The git root if there is one (`git rev-parse --show-toplevel`), otherwise the
current working directory.

## Step 2. Read the last summaries

```
ls <project-root>/Session-Summary/
```

Files are named `session-<project>-<YYYY-MM-DD>.md`, with `-2`, `-3` and so on
for later sessions on the same day.

- **Folder missing or empty:** this is the first recorded session. Say so in
  one line, then orient from what exists: the README and
  `git log --oneline -10`. Do not create the folder, `/end-session` creates it
  when the session closes.
- **Otherwise:** read the 3 most recent, newest first (latest date, then
  highest suffix), or all of them if there are fewer than 3. When they
  disagree, the newest is the source of truth, because it was written with the
  older ones in view. The older two are there to catch anything it missed and
  to show the reasoning behind earlier decisions.

## Step 3. Check the summary against reality

A summary was accurate when it was written. Things may have moved since:
commits made outside a session, files edited by hand, a branch switched.

```
git -C <project-root> log --oneline -10
git -C <project-root> status --short
git -C <project-root> branch --show-current
```

The newest summary records the commit HEAD was at when that session closed.
List anything committed after it, leaving out the summary commits themselves:

```
git -C <project-root> log --oneline <closed-at-commit>..HEAD -- . ":(exclude)Session-Summary"
```

Anything this prints happened outside a recorded session. Say so.

If the summary names a file, a commit or a test count you are about to rely on,
check the current one first. If the project has a quick test command (a
`package.json` test script, pytest, `cargo test`, `go test` and so on), run it
and report the real result rather than the remembered one.

## Step 4. Brief the user

Open with a short briefing, in this order:

1. **Last session finished:** what was done, with commit ids.
2. **Still open:** each item and what blocks it.
3. **Unverified:** anything the last summary flagged as not checked, named one
   by one.
4. **Your notes for this session:** the "Notes for next session" section,
   quoted close to verbatim. These are the user's own words about what to pick
   up next, so do not paraphrase them away.
5. **Changed since:** commits after the last summary, uncommitted files, and
   the current branch.
6. **Starting point:** "Starting from commit `<hash>` on `<branch>`."
   `/end-session` uses this line to find what this session committed.

Then ask what to work on, or, if the notes name a clear next step, offer to
start on it.

Keep it short. This is a briefing, not a re-read of the summary.

## Rules

- A summary is a report, not the truth. Check anything you are about to rely
  on.
- Never say a test passes, a feature works or a file exists without checking
  it in this session.
- Never change files during this skill. It reads and reports only.
