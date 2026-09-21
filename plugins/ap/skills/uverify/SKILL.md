---
name: uverify
description: Use after execute to attack the change and demonstrate how it's broken. Default stance — the change is broken, prove and demonstrate it. Builds an attack checklist (happy-path / negative / invariant hypotheses), runs each freshly, smokes the end-to-end, writes a short summary to the task file's Verify section, loops back to execute on any demonstrated break.
---

# Verify

Try to demonstrate a break in the change. Each check tests a concrete failure hypothesis against current artifacts. Save the outcome in the task file's `## Verify` section, then send failures to `ap:uexecute` or advance to `ap:ureview`.

Read the [brevity rules](../_brevity.md) before responding or writing results. Passed checks take one line. Failures, deferrals, and surprising results carry evidence and reasons.

## Phase 1 — Build the attack list

Build hypotheses from the plan, design constraints, assumptions, and your own inspection. Number them `CK1` through `CKN` within the task file. Keep the working list in-session until writing the results.

Cover these angles:

- Valid input: find a supported input, environment, ordering, or retry that the change mishandles.
- Invalid input: find missing values, type confusion, or malformed input that bypasses validation or raises the wrong error.
- Invariants and assumptions: try to violate each relevant guarantee or disprove an external premise. Reference its design ID.

For example, an item endpoint should reject `name: null` as a client error. Test whether it instead reaches the handler and returns a server error. "Test the happy path" is not a failure hypothesis.

## Phase 2 — Run each attack freshly

Run the attacks during the current verification. Previous-session results do not establish the current state.

- Use minimal direct probes: the direct command, no harness. One-off scripts belong in gitignored, project-local `tmp/` and are removed after use.
- Capture actual commands and output. Inspect what happened before deciding the verdict.
- A failed attempt to break the change is a valid outcome when the probe ran and the evidence supports it. An unavailable probe is a deferral, not evidence that the condition held.
- Respect user authorization for expensive or remote checks. Report missing permission rather than running them or claiming success.

## Phase 3 — Smoke the end-to-end

Run the shortest representative full path: invoke a changed command, call the running endpoint, exercise the interface in a browser, or run a small model step when authorized.

A failing smoke is a demonstrated break. A passing smoke does not replace the other attacks. If it cannot run, name the blocker. Do not substitute a unit test and call it a smoke.

## Phase 4 — Write the Verify summary

Append or replace `## Verify` in the task file, whether the outcome passed or failed.

- `held`: the attack ran and demonstrated no break.
- `broke`: the attack demonstrated a break. Include evidence and expected behavior.
- `deferred`: the attack could not run. Name the blocker and justification.

Set `Result: passed` only when every check held or has a justified, user-visible deferral. Any broken check makes the result `failed`. Never label an unrun check `held`.

```markdown
## Verify

Result: <passed | failed>

Happy-path:
- CK1 — <hypothesis> — <verdict>

Negative:
- CK2 — <hypothesis> — <verdict>

Invariants / assumptions:
- CK3 (IV1) — <hypothesis> — <verdict>
- CK4 (AS1) — <hypothesis> — <verdict>

Smoke: <command and result>
Notes: <failure evidence, expected behavior, deferrals, or reruns>
```

Omit empty categories. Omit `Smoke` if it did not run, but record the blocker in `Notes`. Omit `Notes` only when there is no failure, deferral, or other material evidence to report.

Example failure:

```markdown
## Verify

Result: failed

Negative:
- CK2 — null name on item creation — broke: null reached the handler and raised TypeError, returning 500 instead of 400.

Notes: reject null, empty, and whitespace-only names before the handler runs. Return to execution with this reproduction.
```

## Phase 5 — Handoff

- Every check held or was justifiably deferred: report verification passed, set Status to `reviewing`, and invoke `ap:ureview`, preserving deferrals.
- Any check broke: describe the intended behavior under that input, include the reproduction, and invoke `ap:uexecute`. Do not advance to review.

## Future work and incomplete work

In-scope requirements remain part of this task. Complete them. Dropping one is a scope change, a blocker under `ap:uexecute`'s stop list, never a silent deferral.

A future-work entry requires a design-scope line excluding it or a newly discovered fact that changes scope. Record that justification. Do not relabel a required failing check as future work to pass verification.

## Never

- Claim a check held without running it during the current verification.
- Declare pass when an attack broke.
- Treat a clean diff, lint result, ordinary unit-test result, child assertion, or confidence statement as an adversarial verification by itself.
- Claim there is no way to break the change without naming the angles tried.
- Skip verification or rely on a prior session's verdict.

## Terminal state

The summary is saved. Pass returns to review. Failure returns to execution with evidence and intended behavior.
