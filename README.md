# atpack

`atpack` is an opinionated Claude Code skill pack: plan-driven, minimalistic, fork of [btseytlin/ultrapack](https://github.com/btseytlin/ultrapack) with all git automation removed. Built around frequently clearing context and using one conversation for one feature.

## What's different from ultrapack

ultrapack is "git-centered": it creates branches and worktrees for you, commits incrementally per plan phase, and treats commits as the workflow's artifact. atpack does **none** of that.

In atpack the user owns git:
- You create the branch (or work on `trunk` / `main`) before invoking the workflow.
- The implementer writes code, runs tests, and `git add`s the result. It never commits.
- Between phases you review `git diff --cached` and unstage anything you don't want.
- At the end you create one commit (or amend the existing branch commit) with the staged set.

That's the whole change. Everything else — the task-file-as-source-of-truth flow, the design/plan/execute/verify/review skills, the implementer/reviewer subagents, the brevity rules, the consistency sweeps — is preserved from ultrapack.

The `handsoff` mode and the `git-worktrees` skill are removed (they were tightly coupled to git automation).

## TL;DR

```
/ap:make fix the flaky login test
```

Walks you through design → plan → execute → verify → review. Each stage populates `docs/tasks/<slug>.md`. The task file is the source of truth — any fresh agent can read it and resume from wherever the last one stopped.

Core ideas:
- One file per task. `docs/tasks/<slug>.md` evolves through Design → Plan → Verify → Conclusion.
- Invariants-, principles-, and assumptions-first. Discovered in design, obeyed in plan, checked at review. Short IDs (IV, PC, AS, UK, PH, RK, CK) let later sections reference them without re-quoting.
- Per-phase subagent implementation. Each plan phase dispatched to a fresh `ap:implementer`. Sequential — the user reviews staged changes between phases.
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
- `ap:uexecute` — Dispatch `ap:implementer` per phase (sequential), staging between phases, plan-diff + consistency sweep on the staged diff.
- `ap:uverify` — Positive + negative + invariant checklist, manual smoke test, writes summary to task file, loops back to execute on failure.
- `ap:ureview` — Dispatch `ap:reviewer` subagent: independent review of the merged diff vs branch base, check that all invariants from design and plan still hold.
- `ap:udebug` — Four-phase root-cause investigation.
- `ap:udocument` — Guidance for updating docs, CLAUDE.md, READMEs, in-code comments.

Discipline skills:
- `ap:test-driven-development` — write failing test → make change → test passes.

### Commands

- `/ap:make <description>` — Orchestrate the full flow: task file → design → plan → execute → verify → review → update docs.
- `/ap:try` — Design one positive and one negative test case, run both, report.
- `/ap:step-back` — Circuit breaker: stop, diagnose why approaches failed, propose new direction.
- `/ap:summary` — Produce a summary so another session can continue with zero context.
- `/ap:reflect` — Reflect on the dialogue, extract learnings into CLAUDE.md / memory / docs.

### Agents

- `ap:explorer` (Haiku 4.5) — Codebase tracing, file:line refs, 3–5 essential files.
- `ap:implementer` (Sonnet 4.6) — One phase (or a small bundle): code + tests + `git add` + self-review. Never commits. Fresh context per dispatch.
- `ap:reviewer` (Sonnet 4.6) — Independent review against Plan + Invariants + Assumptions. Confidence-filtered (≥80), severity-tiered.
- `ap:researcher` (Sonnet 4.6) — General-purpose investigation: decompose + systematically answer.
- `ap:summarizer` (Sonnet 4.6) — Drafts the handoff prose for `/ap:summary`; gathers repo state, never writes to disk.

## Credits

This plugin is a fork of [btseytlin/ultrapack](https://github.com/btseytlin/ultrapack) by Boris Tseitlin. The fork strips out the per-phase commit + worktree automation while preserving the rest of the workflow.

## License

WTFPL — see [license.txt](license.txt).
