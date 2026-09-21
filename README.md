# atpack

`atpack` is a Claude Code skill pack for spec-driven work: one task file drives design, plan, execute, verify and review. Fork of [btseytlin/ultrapack](https://github.com/btseytlin/ultrapack).

## Git contract

The user owns git:
- You create the branch (or work on `trunk` / `main`) before invoking the workflow.
- The implementer writes code, runs tests, and `git add`s the result. It never commits.
- At the end you create one commit (or amend the existing branch commit) with the staged set.

## Kinds

`/ap:make` classifies the request without asking and writes `**Kind:**` into the task file:
- `change` — design, plan, execute, verify, review. Design and Plan each get one approval; from Execute to Finish nothing asks.
- `answer` — a question or investigation. Design holds the answer, the task ends at `done` with no Plan or Verify.
- `review` — `review <MR|branch>`. `ap:ureview <ref>` reviews the ref read-only and leaves fixes to the author.

A `/ap:make` on a `done` task with new arguments starts `docs/tasks/<slug>-2.md` as a follow-up.

## TL;DR

```
/ap:make fix the flaky login test
```

Walks you through design → plan → execute → verify → review. Each stage populates `docs/tasks/<slug>.md`. The task file is the source of truth — any fresh agent can read it and resume from wherever the last one stopped.

Core ideas:
- One file per task. `docs/tasks/<slug>.md` evolves through Design → Plan → Verify → Conclusion.
- Invariants-, principles-, and assumptions-first. Discovered in design, obeyed in plan, checked at review. Short IDs (IV, PC, AS, UK, PH, RK, CK) let later sections reference them without re-quoting.
- Per-phase subagent implementation. Each plan phase dispatched to a fresh `ap:implementer`. Sequential, one shared git index.
- Mandatory manual testing. Agent must run what it built before claiming done.
- The user owns git. The workflow stages, never commits.

## Install

```
/plugin marketplace add ~/.claude/plugins/marketplaces/atpack/
/plugin install ap@atpack
```

Then `/reload-plugins`. Verify with `/ap:make` or by listing skills.

## Design

Each stage of task planning and execution is a skill. `/ap:make` is a helper command that orchestrates the whole flow.

`ap:udesign` is the first stage: discuss trade-offs with the user, discover invariants (specific things that must hold, e.g. "class Player must not access internals of class Enemy"), principles (softer guidance, like "prefer composition over inheritance"), assumptions (unverified premises the design rests on — the Conclusion reports whether each held), and unknowns (open questions to resolve during plan/execute). Prepare initial spec in the task file.

`ap:uplan` populates the task file with a specific plan. Define what files to change, what classes and methods to update or create, what interfaces they will have, what is the test strategy, break down into phases, define order of execution. No code blocks unless they are especially tricky.

`ap:uexecute` dispatches `ap:implementer` agents per plan phase. Each phase: implementer writes code, runs tests, runs `git add`, reports back. The dispatcher then runs a plan-diff check (`git diff --cached`) and a consistency sweep before moving to the next phase. Uses TDD for tasks where it's helpful. Sequential only — no parallel batching.

`ap:uverify` performs manual smoke testing, defining a checklist of fast positive (what should work) and negative (what should not work) checks based on invariants. Writes summary to task file. Loops back to execute on failure.

`ap:ureview` dispatches an independent `ap:reviewer` subagent that sees the design, plan, and the merged diff (committed + staged + unstaged vs branch base). It doesn't know the implementer's rationale. The dispatcher then processes findings fairly and writes the Conclusion.

Finally, the conclusion section of the task markdown file is populated. The user is then free to commit / amend / push as they choose.

## Details

### Skills

Process skills:
- `ap:udesign` — Brainstorm requirements, populate Design + Invariants + Principles + Assumptions + Unknowns, decide whether to use TDD.
- `ap:uplan` — Plan: what files to change, what class/methods and with what interfaces, test strategy, order. Only non-trivial code blocks.
- `ap:uexecute` — Dispatch `ap:implementer` per phase (sequential, shared index), stages each phase, plan-diff + consistency sweep on the staged diff.
- `ap:uverify` — Positive + negative + invariant checklist, manual smoke test, writes summary to task file, loops back to execute on failure.
- `ap:ureview` — Dispatch `ap:reviewer` subagent: independent review of the merged diff vs branch base, check that all invariants from design and plan still hold.
- `ap:udebug` — Four-phase root-cause investigation.
- `ap:udocument` — Guidance for updating docs, CLAUDE.md, READMEs, in-code comments.

Discipline skills:
- `ap:test-driven-development` — write failing test → make change → test passes.
- `skills/_brevity.md`, `skills/_principles.md` — writing rules and global principles (GPC1–GPC8) every skill reads first.

### Commands

- `/ap:make <description>` — Orchestrate the full flow: task file → design → plan → execute → verify → review → update docs.

### Agents

- `ap:explorer` — Codebase tracing, file:line refs, 3–5 essential files.
- `ap:implementer` — One phase (or a small bundle): code + tests + `git add` + self-review. Never commits. Fresh context per dispatch.
- `ap:reviewer` — Independent review against Plan + Invariants + Assumptions. Confidence-filtered (≥80), severity-tiered.
- `ap:researcher` — General-purpose investigation: decompose + systematically answer.

## Porting from upstream

The repo has `upstream` = btseytlin/ultrapack. To port a change:

```bash
git log upstream/main --since=<date of the last port> --oneline
git diff upstream/<sha>:<file> plugins/ap/<file>        # plugin is the repo root upstream
git show upstream/main:<file> | sed 's/up:/ap:/g' > plugins/ap/<file>   # only for files the fork never edited
```

Claude Code loads the plugin from `~/.claude/plugins/cache/atpack/ap/<version>/`, not from this directory. After a
commit, reinstall and check the cache matches:

```bash
claude plugin update ap@atpack        # then restart Claude Code
diff -rq ~/.claude/plugins/cache/atpack/ap/$(python3 -c "import json;print(json.load(open('plugins/ap/.claude-plugin/plugin.json'))['version'])") plugins/ap
```

## Credits

This plugin is a fork of [btseytlin/ultrapack](https://github.com/btseytlin/ultrapack) by Boris Tseitlin. The fork strips out the per-phase commit + worktree automation while preserving the rest of the workflow.

## License

WTFPL — see [license.txt](license.txt).
