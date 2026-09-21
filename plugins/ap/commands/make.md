---
description: Orchestrate the full atpack workflow — slug, task file, design, plan, execute, verify, review. Size-aware, resume-ready. The user owns the branch and commits; the workflow only stages.
---

# /ap:make

Drives a task through the full atpack workflow: one task file at `docs/tasks/<slug>.md`, evolving through Design → Plan → Conclusion. Each stage is a separate skill. You orchestrate; the skills do the work.

The user manages git themselves: they create the branch (or work on `trunk` / `main`), they decide when to commit and amend. This workflow never runs `git commit`, `git checkout`, `git branch`, or `git worktree`.

<required>
Before responding or writing the task file, read `${CLAUDE_PLUGIN_ROOT}/skills/_brevity.md` and `${CLAUDE_PLUGIN_ROOT}/skills/_principles.md`. Apply them throughout the workflow.
</required>

## Task file header

The header is the state machine every skill reads on entry. Exactly one writer per transition:

- `**Kind:**` — `change` | `answer` | `review`, written by this command in step 3/4, read by `ap:udesign`, `ap:ureview` and the Rules below.
- `**Status:**` — `design` (this command, on create) → `planning` (`ap:udesign`) → `executing` (`ap:uplan`) → `verifying` (`ap:uexecute`) → `reviewing` (`ap:uverify`) → `done` (`ap:ureview`, or `ap:udesign` for Kind `answer`). A skipped stage's status is written by this command when it skips.

## Arguments

The user's description of the task follows the command. May be a one-liner ("fix the flaky login test") or a paragraph. Use it as the seed for the slug and the initial framing for `ap:udesign`.

## Flow

### 1. Slug

Derive a kebab-case slug from the description, 3 words max (e.g. "flaky-login-test"), and proceed.

### 2. Resume check

Before creating a new task file, check if `docs/tasks/<slug>.md` already exists.

- Exists: read `**Status:**` from the header. Resume from the next stage:
  - `design` → continue design
  - `planning` → run `ap:uplan`
  - `executing` → run `ap:uexecute`
  - `verifying` → run `ap:uverify`
  - `reviewing` → run `ap:ureview`
  - `done` with new arguments → this is a follow-up. Create `docs/tasks/<slug>-2.md` (next free number) from the template with the first line under the title "Follow-up of `<slug>.md`", append `- Follow-up: <slug>-2.md (YYYY-MM-DD)` to the original's `## Conclusion`, and run the normal flow in the new file
  - `done` without arguments → print the Conclusion
- Doesn't exist: proceed to step 3.
- Multiple in-flight tasks: if more than one `docs/tasks/*.md` has Status ≠ `done`, list them and ask which one the user means (or whether this is a new task).

### 3. Create task file

Create `docs/tasks/<slug>.md` from the template. Status = `design`. Write the resolved Kind (step 4) into the `**Kind:**` line, never the placeholder.

Template:

```markdown
# <Task Title>

**Status:** design
**Kind:** <change | answer | review>

## Context
<empty — filled by ap:udesign: observations about how things are now that motivate the design>

## Design
<empty — filled by ap:udesign>

### Invariants
<empty — IV1, IV2, … : hard constraints that must hold>

### Principles
<empty — PC1, PC2, … : softer guidance>

### Assumptions
<empty — AS1, AS2, … : unverified premises the design rests on; conclusion must report whether each held>

### Unknowns
<empty — UK1, UK2, … : open questions left to plan / execute; conclusion must report whether each resolved>

## Plan
<empty — filled by ap:uplan>

## Verify
<empty — filled by ap:uverify>

## Code smells
<empty — file:line + one-line smell passed while exploring and left unfixed; deleted if none>

## Conclusion
<empty — filled by ap:ureview>
```

### 4. Kind and size

Classify the Kind from the request yourself, state it in the first reply, and write it into the task file's `**Kind:**` line before invoking any skill:

- `answer` — the request is a question, or says "don't change" / "just answer" / "investigate" / "explain".
- `review` — the request says "review <MR|branch>" or names a colleague's branch.
- `change` — everything else.

For Kind `change`, also classify size:

- Trivial — one-line change, typo, rename. Skip Design and Plan (set Status `executing`). Go straight to Execute. Status file still created.
- Small — single file or single concept change. Skip Design (set Status `planning`). Plan runs.
- Medium / Large — full flow.

Confirm the classification with the user before skipping any stage. When unsure, ask.

### 4a. Kind `answer`

Invoke `ap:udesign` in research mode. `## Design` holds the answer, and `ap:udesign` writes the Conclusion, deletes the `## Plan` and `## Verify` headers, and sets Status `done`. Then go to Finish — no Plan, Execute, Verify, or Review.

### 4b. Kind `review`

Delegate to `ap:ureview <ref>`. It reuses the task file from step 3, writes Design = review scope and the questions asked, deletes the `## Plan` and `## Verify` headers, stays read-only, writes severity-tiered findings to `## Conclusion`, and sets Status `done`. On a colleague's branch the fixes are left to the author. Then go to Finish.

### 5. Design stage (unless skipped)

Invoke `ap:udesign`. It populates `## Context`, `## Design`, `### Invariants` (IV), `### Principles` (PC), `### Assumptions` (AS), `### Unknowns` (UK), records `TDD: yes / no (reason)`, and sets Status `planning`.

### 6. Plan stage (unless skipped)

Invoke `ap:uplan`. It populates `## Plan` and sets Status `executing`.

### 7. Execute stage

Invoke `ap:uexecute`. Implements the plan and stages each phase. Runs to Verify without stopping.

### 8. Verify loop

Invoke `ap:uverify`. On failure: `ap:uverify` describes how each failure *should* have worked, control returns to `ap:uexecute`. Loop until verify passes.

### 9. Review stage

Invoke `ap:ureview`. It dispatches `ap:reviewer`, processes findings, and fills `## Conclusion`.

Once the task is concluded as `done`, run the docs-refresh check (see below).

From Execute to Finish the flow does not end the turn or ask. It stops only on the blockers listed in `ap:uexecute`.

### 10. Finish

Grep `**Status:** done` in the task file and set it if it is absent. Print a summary of what's staged (`git diff --cached --stat`) and remind the user that committing, amending, and pushing are theirs to do. The workflow is complete.

## After task is done — docs refresh

Run this once, after Review concludes the task as `done` (not after every stage). Scan the project docs and update them if the work surfaced something they should reflect. Cheap, light-touch; not a full doc pass.

Files to scan:
- `CLAUDE.md` (project-wide agent guidance)
- `README.md`
- `docs/**/*.md` (project documentation, excluding the task file itself and archived tasks)

What to look for:
- New conventions, invariants, or principles that should be global → update `CLAUDE.md`
- New components, commands, or features the README should mention
- Stale content contradicted by the stage's work → delete or correct
- Pointers to the task file if future contributors would benefit

Rules:
- If nothing needs updating: say so in one line and move on. Do not invent edits.
- If updates are needed: make them directly, then summarize what changed in 1-3 lines (e.g. "README: fixed install instructions; CLAUDE.md: no change"). Do not prompt for approval first. Do not produce a detailed diff — the user will git-diff if they want.
- Follow `ap:udocument`: lead with why, lists over tables, no aspirational content, kill stale content.
- Do not duplicate content across task file and project docs — pick one home per fact.

## Stop conditions

Stop and ask the user when:

- Size classification is genuinely unclear
- User has expressed a preference (scope, TDD) that conflicts with the auto-inference
- Any stage's skill returns a blocker

## Rules

- Never skip Review for Kind `change`
- Never run `git commit`, `git commit --amend`, `git checkout`, `git branch`, `git worktree`, or `git push`. The user owns all branch and commit operations
- Keep the task file as the single source of truth — each stage reads it, each stage writes to it
- External spec / design docs (e.g. anything under `docs/specs/`) are read-only during execute. If a stage finds the spec is wrong, surface it to the user — don't mutate it silently
- Don't assume prior session memory — the next agent may be a fresh context reading only the task file

## Terminal state

Task file Status = `done`, Conclusion filled, user has been shown the staged-diff summary.
