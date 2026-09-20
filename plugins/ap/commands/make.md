---
description: Orchestrate the full atpack workflow — slug, task file, design, plan, execute, verify, review. Size-aware, resume-ready. The user owns the branch and commits; the workflow only stages.
---

# /ap:make

Drives a task through the full atpack workflow: one task file at `docs/tasks/<slug>.md`, evolving through Design → Plan → Conclusion. Each stage is a separate skill. You orchestrate; the skills do the work.

The user manages git themselves: they create the branch (or work on `trunk` / `main`), they review staged changes between phases, they decide when to commit and amend. This workflow never runs `git commit`, `git checkout`, `git branch`, or `git worktree`.

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
  - `reviewing` → run `ap:ureview`
  - `done` → ask the user what they want to do (start a follow-up, re-open, view conclusion)
- Doesn't exist: proceed to step 3.
- Multiple in-flight tasks: if more than one `docs/tasks/*.md` has Status ≠ `done`, list them and ask which one the user means (or whether this is a new task).

### 3. Create task file

Create `docs/tasks/<slug>.md` from the template. Status = `design`.

Template:

```markdown
# <Task Title>

**Status:** design

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

## Conclusion
<empty — filled by ap:ureview>
```

### 4. Size classification

Based on the task description, classify size:

- Trivial — one-line change, typo, rename. Skip Design and Plan. Go straight to Execute. Status file still created.
- Small — single file or single concept change. Skip Design. Plan runs.
- Medium / Large — full flow.

Confirm the classification with the user before skipping any stage. When unsure, ask.

### 5. Design stage (unless skipped)

Invoke `ap:udesign`. It populates `## Design`, `### Invariants` (IV), `### Principles` (PC), `### Assumptions` (AS), `### Unknowns` (UK), and records `TDD: yes / no (reason)`. Status → `planning`.

### 6. Plan stage (unless skipped)

Invoke `ap:uplan`. It populates `## Plan`. Status → `executing`.

### 7. Execute stage

Invoke `ap:uexecute`. Implements the plan, staging changes after each phase for the user to review. The user is expected to inspect `git diff --cached` between phases and unstage anything they don't want.

### 8. Verify loop

Invoke `ap:uverify`. On failure: `ap:uverify` describes how each failure *should* have worked, control returns to `ap:uexecute`. Loop until verify passes.

### 9. Review stage

Status → `reviewing`. Invoke `ap:ureview`. It dispatches `ap:reviewer`, processes findings, fills `## Conclusion`. Status → `done`.

Once the task is concluded as `done`, run the docs-refresh check (see below).

### 10. Finish

Print a summary of what's staged (`git diff --cached --stat`) and remind the user that committing, amending, and pushing are theirs to do. The workflow is complete.

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

- Never skip Review
- Never run `git commit`, `git commit --amend`, `git checkout`, `git branch`, `git worktree`, or `git push`. The user owns all branch and commit operations
- Keep the task file as the single source of truth — each stage reads it, each stage writes to it
- External spec / design docs (e.g. anything under `docs/specs/`) are read-only during execute. If a stage finds the spec is wrong, surface it to the user — don't mutate it silently
- Don't assume prior session memory — the next agent may be a fresh context reading only the task file

## Terminal state

Task file Status = `done`, Conclusion filled, user has been shown the staged-diff summary.
