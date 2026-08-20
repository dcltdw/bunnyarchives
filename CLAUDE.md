# CLAUDE.md

Guidance for working in this repo. See [README.md](README.md) for the
user-facing description.

## What this is

An opinionated, generic campaign LARP system following the "everyone has the
same number of skills" model pioneered by Attaway.

The repo is currently a bootstrap: license, README, changelog, and this file.
Design of the system itself has not started, and gets its own spec and plan
when it does.

## Collaboration

Universal collaboration rules — branch → PR → wait for approval, many small
single-purpose PRs, the `Co-Authored-By` model stamp on commits, secret-scan
before push, project-board flow, and verify-before-done — come from the
machine-global import in `~/.claude/CLAUDE.md` (`@~/.claude/dcltdw/AGENTS.md`),
so they are not duplicated here. Edits to that shared file propagate
automatically; this file only adds repo-specific deltas on top.

There are no repo-specific deltas yet.

## Project board

- Board: https://github.com/users/dcltdw/projects/9 (`PVT_kwHOAAdfes4Bg-y1`) —
  **private**.
- Status field `PVTSSF_lAHOAAdfes4Bg-y1zhf8RhY`:
  Todo `830b6b4f`, In Progress `d4f224e7`, Done `dfe94e4e`,
  Won't Do `7f0fdb2d`

Board items are real GitHub issues; a PR that resolves one should use
`Closes #N` so the merge auto-closes the issue. Moving the board card to Done
is still a separate manual step — `Closes #N` closes the *issue*, and the
board Status field is independent of issue open/closed state.

Re-derive the IDs if they drift:

```sh
gh project field-list 9 --owner dcltdw --format json
```
