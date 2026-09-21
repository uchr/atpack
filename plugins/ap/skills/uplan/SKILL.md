---
name: uplan
description: Use after design to turn a validated spec into a lean implementation plan. Outputs the `## Plan` section of the task file — files, line numbers, class/method names, invariants, test strategy, order. Ends with a scope-creep / simpler-way check.
---

# Plan

Convert the approved design into an implementation contract for `ap:uexecute`. Fill `## Plan` in the existing `docs/tasks/<slug>.md`. Do not create a separate plan file.

Read the [brevity rules](../_brevity.md) and [global principles](../_principles.md) before planning. Preserve required content and omit optional sections with no useful content. Deviations, deferrals, failures, and risks retain evidence and reasons.

## What a plan is

A plan is the minimum information an engineer needs to implement the design without re-doing the thinking:

- Concrete locations: exact file paths, line ranges, class/method/interface names. "Modify `parser.py`" is not enough — `parser.py:120-160, Parser.tokenize()` is.
- Per-file change bullets: what changes in each file, by name. Not prose.
- Signatures for new or changed functions and methods. No bodies.
- Phases as staging units: ordered phases, each producing a coherent staged set.
- Test strategy: the behaviors to cover, not test code.
- Snippets only where words fail: one regex, one tricky algorithm, one non-obvious API call.

If an engineer could read your plan and implement the wrong thing, the plan is incomplete. If they need to re-derive the design to understand the plan, the plan is bloated.

## Scope

Design decides the outcome and scope. Planning names the concrete changes needed to reach it. If a new finding requires splitting the task, changing the design, or expanding scope, stop and ask the user. Do not silently rewrite the goal.

## Process

<required>
1. Read context, design, invariants, principles, assumptions, and unknowns in the task file.
2. Identify files, symbols, and interfaces to change. Include only defects within the approved scope in the plan. Report unrelated findings under `## Code smells` without adding their fixes.
3. Divide the work into coherent phases. Each phase is one staged set the user can review in one go.
4. Write concrete file changes, signatures, verification behavior, and dependencies. Follow the design's test-driven-development decision.
5. Map compatibility risks to phases and mitigation steps. If a new break was not covered by the design, stop and ask the user. For greenfield work, state that this check does not apply.
6. Review the plan for coverage, correctness, and simplicity using the checks below.
7. Present the plan and wait for approval, set Status `executing`, then invoke `ap:uexecute`.
</required>

## Required contents

Always include:

- Approach and why it fits the design.
- Each changed or new file as `path:lineA-lineB`, with affected symbols and concrete changes. For example, name `Parser.tokenize()` rather than just "modify parser".
- New or changed signatures without implementation bodies.
- Ordered phases, each with the staged set it produces.
- References to the design constraints each phase preserves or relies on.

Include when relevant:

- Test strategy: behaviors to cover, with failing tests first when `TDD: yes`.
- Dependencies, open questions, risks, and rollback.
- A snippet only when prose cannot express the most critical detail in a phase. Do not write full implementations in the plan.

## ID conventions

- `PH1`, `PH2`, and so on identify phases. Each phase heading is `### PH<N> — <name>`.
- `RK1`, `RK2`, and so on identify risks, each defined in one sentence.
- Reference existing design entities by their IDs rather than repeating definitions.

## Format

```markdown
## Plan

Approach: <strategy and reason>

### PH1 — <name>
- 1.1 `path/to/file.ext:lineA-lineB` (create|modify)
  - `Class.method(arg: Type) -> Ret` — <concrete change>
  - Respects: IV2, AS1
- Stage: <one-line summary of what gets staged for review>

### Test strategy
- <behavior and affected file or phase>

### Order & dependencies
- <what blocks what>

### Risks / rollback
- RK1 — <risk and mitigation>
```

Repeat phase entries as needed. Omit irrelevant optional sections.

## Self-review

<required>
1. Every invariant, principle, assumption, and unknown maps to a plan bullet or an explicit deferral to execution or verification.
2. File paths and symbols are concrete and consistent. Remove placeholders such as "handle edge cases" or "similar to phase 1".
3. Every phase follows the global principles. A deviation names the principle and the reason.
4. The test strategy and compatibility mitigations match the design.
5. Keep the plan proportional to the work. Fix these issues inline without a review subagent.
</required>

## Final check — scope creep, elegance, simpler way

<required>
- Does each phase serve the approved design, without unrelated refactors, extra validation layers, or speculative abstractions?
- Can phases or duplicate paths be combined without losing clear ownership?
- Would a simpler approach meet the goal with less work? Present a material alternative to the user rather than changing scope silently.
</required>

Revise a bloated plan before handoff. For example, remove a generalization phase whose only justification is a second caller that does not exist.

## Terminal state

The plan is written, self-reviewed, scope-checked, and explicitly approved before execution begins.
