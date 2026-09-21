---
name: uexecute
description: Use to implement an approved plan. Dispatches the ap:implementer agent per phase. Runs plan-diff check on the staged diff and consistency sweep after each phase. Forbids silent fallbacks and mutation of external spec/design docs. Dispatches the planner skill when deviations invalidate the plan.
---

# Execute

Implement the approved `## Plan` from `docs/tasks/<slug>.md`. You are the dispatcher — each phase is handed to a fresh `ap:implementer` subagent. The implementer writes code, runs tests, and stages the changes (`git add`) — it never commits. After each phase returns, you inspect the staged diff and run the plan-diff + consistency checks before moving on. The user decides what to commit and owns the branch.

## Before starting

<required>
1. Read the full task file: context, design, invariants, principles, assumptions, unknowns, and plan.
2. Raise ambiguity, missing dependencies, or contradictory instructions before editing.
3. Create a checklist entry per phase with TodoWrite.
4. Read the [brevity rules](../_brevity.md) before writing task progress or responding. Omit empty deviations and risks. Deferrals name the needed path, command, or decision and include evidence and a reason.
</required>

## Per-phase loop

Execute phases in order. Phases run one at a time: two implementers calling `git add` at once race on `.git/index.lock`.

For each phase:

<required>
1. Mark the phase `in_progress` in TodoWrite.
2. Dispatch the implementer — see "Dispatch per phase" below.
3. On implementer return, handle status:
   - `DONE` → continue to step 4.
   - `DONE_WITH_CONCERNS` → read the concerns. Resolve or record them, then continue.
   - `NEEDS_CONTEXT` → supply missing context, re-dispatch.
   - `BLOCKED` → diagnose. If the plan is wrong, invoke `ap:uplan`. If context-only, re-dispatch. Do not retry identically.
4. **Plan-diff check.** Run `git diff --cached -- <files the implementer touched>`. For every plan bullet in this phase: is it reflected in the staged diff? For every change in the staged diff: is it covered by a bullet, or by a deviation the implementer reported? Any unreported structural gap → record as a deviation, or re-dispatch with correction.
5. **Consistency pass.** Verify the implementer's claimed sweep rather than repeating it blindly. If they missed siblings, either re-dispatch or fix inline — the staged set is updated in place.
6. Mark the phase `completed`.
</required>

## Dispatch per phase

Each phase runs in a fresh `ap:implementer` subagent. You are the dispatcher.

<required>
Start every child prompt with the authorization and guidance fields below, then the phase verbatim and its design constraints. Resolve guidance paths for this machine rather than copying another user's paths. Require the child to read that guidance before acting and to stop if it is unavailable.

```text
Authority: <read-only or approved editing scope>
Guidance: <absolute paths to applicable global and project instructions and guides>
Limits: <approved commands, installations, spending, duration, and restart permissions>
Precedence: user authorization and supplied guidance override conflicting workflow advice
Phase: <verbatim phase text from the plan>
Invariants: <definitions from design>
Principles: <definitions from design>
Assumptions: <definitions from design>
TDD: <yes | no (reason)>
Working directory: <absolute path>
```

Pass the design rationale and constraints the phase depends on. Omit unrelated session history, prior-phase chatter, later phases, and the full task file.

Implementers edit only the phase's paths, run their checks, stage with `git add`, and report status, changed files, checks and results, concerns, and deviations. They never commit.
</required>

**When to skip dispatch and do it inline:**
- Trivial phase (typo, one-line import fix, changelog edit)
- Phase needs mid-work interactive user input
- A phase-N-and-N+1 fix follows from review findings, small enough to just edit

When inlining, you (the dispatcher) still stage the result with `git add` and never commit.

## Test-driven development

If design records `TDD: yes`, follow `ap:test-driven-development`: observe a failing test, implement the minimum change, and refactor with tests green. If `TDD: no`, skip test-first development and verify through `ap:uverify`.

## Supporting investigation

Use an [explorer](../../agents/explorer.md) when missing code context, a call graph, or essential-file discovery exceeds a quick local read or search. Use a [researcher](../../agents/researcher.md) for external information requiring synthesis across sources. A single documentation lookup does not need delegation.

Pass a tight question or scope and the absolute working directory. For research, include relevant subquestions and source or depth limits.

## Consistency pass

When tightening a rule, renaming a symbol, or changing a pattern, search the staged diff (`git diff --cached`) and the wider repository (`git grep`) for siblings. Check the introducing commit when it helps locate them. Apply the same in-scope change in the same staged set, or record a justified deferral.

For example, if a metric now rejects empty input, check sibling metrics for the same silent fallback.

## Code smells

Follow the [global principles](../_principles.md#incidental-code-smells): fix only defects within the approved request. Record unrelated smells under `## Code smells` as `file:line — issue` without changing them. Owning a file does not authorize incidental cleanup inside it.

## External specs and deviations

External specs, including `docs/specs/`, are read-only inputs. A wrong spec is a blocker (see When to stop and ask): stop and let the user choose whether to revise it, revise the plan, or accept a deviation.

Never edit the approved plan to hide what changed. Record a deviation under `## Conclusion → ### Deviations from plan` as `what changed — why`, with evidence.

- Minor changes, such as a helper rename or a small reorder, can continue when later phases remain valid.
- Structural changes that invalidate later phases stop execution. Give `ap:uplan` the completed work, invalidated steps, and new facts so it can revise the plan through its approval process.

## No silent fallbacks

Do not invent defaults, swallow exceptions, or return placeholder success. If required data is missing and the plan defines no fallback, fail loudly. Tell the user immediately and record the failure point under `## Conclusion → Known risks`.

For a required timeout, use `config["timeout"]`, not `config.get("timeout", 30)`. If partial configuration is possible, record that it raises and ask whether the contract needs explicit validation or an approved default.

## When to stop and ask

These are the only stops. Progress between phases is written to the task file, not asked.

- A plan instruction is ambiguous or self-contradictory
- An external spec the plan relies on is wrong
- A test fails in a way that suggests the plan is wrong (not just the implementation)
- Verify would obviously fail even after you finish
- You're about to invent a fallback / default / catch-all

Don't force through. Ask.

## Never

- Run `git commit`, `git commit --amend`, `git reset`, `git checkout`, `git branch`, `git worktree`, or `git push`. The user owns all of these
- Claim complete without running what you built
- Edit the Plan section to hide a deviation
- Edit an upstream spec file during execute (flag issues, don't mutate)
- Invent a silent fallback to avoid stopping
- Move on to the next phase without running the plan-diff check and consistency pass

## Terminal state

All phases done and staged → set Status to `verifying`, then invoke `ap:uverify`. Do not skip to review. Do not commit anything yourself.
