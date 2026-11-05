# Session Summary skills

Two [Claude Code](https://claude.com/claude-code) skills that let you close a
work session and pick it up later without explaining everything again.

- **`/end-session`** writes a summary of the session into a `Session-Summary/`
  folder in your project, asks you for notes for next time, commits that one
  file, and tells you when everything is saved and it is safe to close.
- **`/start-session`** reads the last summaries back, checks them against your
  current git state, and briefs you: what was finished, what is still open,
  what was never verified, and the notes you left for yourself.

No services, no database, no API keys. Just markdown files in your repo, and
git.

## Install

Clone the repo and copy the two skill folders into your Claude Code skills
directory.

macOS / Linux:

```bash
git clone https://github.com/Stel777/Session-Summary-skills.git
cp -r Session-Summary-skills/start-session Session-Summary-skills/end-session ~/.claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/Stel777/Session-Summary-skills.git
Copy-Item -Recurse Session-Summary-skills\start-session, Session-Summary-skills\end-session "$HOME\.claude\skills\"
```

To use them in one project only, copy them into `<project>/.claude/skills/`
instead. Then start a new Claude Code session so the skills are loaded.

## Use

At the start of a session:

```
/start-session
```

When you are done:

```
/end-session
```

That is the whole workflow. The first time you run `/end-session` in a
project, it creates `Session-Summary/`. From then on, every `/start-session`
reads the last three summaries.

## What a summary contains

One file per session, named `session-<project>-<YYYY-MM-DD>.md` (`-2`, `-3`
for later sessions on the same day):

| Section | What goes in it |
|---|---|
| Header | Project, date, agent, branch, the commit the session started and closed at |
| Task plan | What the session set out to do, each item done, partial or not started |
| Findings | What was learned or decided, with the evidence |
| Progress | What was done, each item with its commit id, and real test results |
| Still open | Each open item and what blocks it |
| Unverified | Anything built or claimed but not checked. Always present, "None" is valid |
| Notes for next session | Your own closing notes, close to word for word |

Each summary stands alone. Open items from the previous summary are carried
forward and marked done, still open or dropped, so the newest file is always
the full picture.

## What it commits

Only the summary file, using `git commit -- <path>`, so anything else you have
staged or modified stays exactly as it was. It never pushes.

It skips the commit, and tells you why, when:

- the project is not a git repo,
- a merge, rebase, cherry-pick or revert is in progress,
- HEAD is detached,
- `Session-Summary/` is gitignored, or
- a pre-commit hook rejects the commit.

The summary file is written either way.

**If your repo is public:** summaries are committed like any other file, so
they become public when you push. To keep them local, add `Session-Summary/`
to your `.gitignore`. The file is still written, and the commit is skipped.

## Why

A new session starts with no memory of the last one. The usual fixes are
explaining everything again or keeping notes by hand. These skills make
note-taking part of closing the session. The summary is written from what git
actually shows rather than from what anyone remembers, and its "Still open"
list carries forward until each item is actually done.

## License

[MIT](LICENSE)
