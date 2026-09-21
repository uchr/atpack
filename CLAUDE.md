# atpack

Claude Code plugin for spec-driven development. The user owns git: no commits, branches or worktrees from any skill, the workflow only `git add`s. Fork of [btseytlin/ultrapack](https://github.com/btseytlin/ultrapack).

## Repo layout

- `.claude-plugin/marketplace.json` — marketplace manifest (lists `ap`)
- `plugins/ap/.claude-plugin/plugin.json` — plugin manifest
- `plugins/ap/{skills,commands,agents,hooks}/` — plugin contents
- `docs/tasks/*.md` — historical task files inherited from ultrapack (design notes; not runtime artifacts)
- `README.md`, `CLAUDE.md` — repo docs

Everything under `plugins/ap/` loads into Claude Code. Everything outside (`docs/`, README, CLAUDE.md) is repo-only.

## Naming

Internal plugin name: `ap`. Slash/skill invocations use the `ap:` prefix: `/ap:make`, `ap:udesign`, `ap:reviewer`. Process skills are `u`-prefixed (`udesign`, `uplan`, `uexecute`, `uverify`, `ureview`, `udebug`, `udocument`) — inherited from ultrapack to dodge collisions with Claude Code built-ins.

## Design principles

- **Minimal** — only skills we actually use; no speculative additions
- **Doc-only** — no runtime code, no unit tests; verification is install-and-invoke
- **The user owns git** — no `git commit`, `git checkout`, `git branch`, `git worktree`, or `git push` is run by any skill or agent. Ever. Implementer stops at `git add`.

## Versioning

Plugin version lives in `plugins/ap/.claude-plugin/plugin.json`. Always bump the patch digit (`x.y.Z`) when merging, finalizing, or otherwise landing changes on the working branch. Default to patch; ask before bumping minor (`x.Y.z`) or major (`X.y.z`).
