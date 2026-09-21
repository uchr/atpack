---
name: ureview
description: Use after verify passes for the future maintainer's audit — sit in the chair of the person who'll touch this code in 3-6 months and ask "what will bite us later?" at the decision level. Surfaces wrong abstractions, load-bearing-but-unobvious shapes, next-change traps, drift from surrounding code. Raises a Scope flag if the whole change looks like the wrong call. Dispatches ap:reviewer for an independent, critical review and fills the task file's `## Conclusion`.
---

# Review

Find choices that will make the next change difficult: wrong abstractions, implicit constraints, checks in the wrong layer, or drift from surrounding code. Question the scope when the completed work exposes a wrong premise, but leave redesign to the user and `ap:udesign`.

In the full `/ap:make` workflow, run after verification passes, regardless of task size. Save the outcome in `## Conclusion`.

## Standalone review (Kind `review`)

Invoked as `ap:ureview <ref>` with a branch or MR ref. The task file exists from `/ap:make` step 3 (create it from the template if invoked directly): set Kind `review`, write `## Design` as the review scope and the questions asked, delete the `## Plan` and `## Verify` headers, and review the ref's own diff instead of the working tree.

```bash
BASE=$(git merge-base <ref> trunk 2>/dev/null || git merge-base <ref> main)
git diff "$BASE" <ref>
```

Never edit a file or `git add` in this kind — on a colleague's branch the fixes are left to the author. `## Conclusion` holds the severity-tiered findings, and Status becomes `done`. The working-tree `BASE` block in step 1 belongs to Kind `change`.

## Brevity

Read the [brevity rules](../_brevity.md). Keep the outcome to one sentence. Omit empty findings, deviations, risks, scope flags, and future work. Retain invariant evidence and assumption or unknown outcomes whenever the task defined them.

Keep `## Code smells` separate from the conclusion. Delete it if empty. Promote a recorded smell to future work only when scheduling its fix, without duplicating the entry.

## Independent reviewer and dispatcher

The `ap:reviewer` subagent is critical and reports concrete issues at confidence 80 or above, classified by severity. You (the dispatcher) are fair: verify each finding against the codebase before accepting or rejecting it. A tough reviewer catches more real issues; a fair dispatcher avoids overcorrecting on mistaken calls. Do not replace independent review with your own inspection.

## Process

### 1. Dispatch `ap:reviewer`

The user manages git themselves and may have any combination of (committed on this branch) + (staged) + (unstaged) changes that together make up the task scope. Compute the full review diff against the branch point.

Run every git command from the current working directory — do **not** `cd` to the repo root. `git` resolves the enclosing repo from any subdir, and the user's `pwd` may be a monorepo subproject whose tooling (bazel targets, npm scripts, IDE markers) resolves paths relative to that cwd. Changing dir breaks that tooling for follow-up commands.

```bash
# Run from cwd; do not cd to git toplevel.
# Pick the most likely base — the one that exists locally.
BASE=$(git merge-base HEAD trunk 2>/dev/null \
    || git merge-base HEAD main 2>/dev/null \
    || git merge-base HEAD master 2>/dev/null)
# Full task scope: committed + staged + unstaged, all vs branch point.
git diff "$BASE"
```

Dispatch the `ap:reviewer` agent with:
- Task file path (`docs/tasks/<slug>.md`)
- The `BASE` ref (so the reviewer can re-derive the diff if needed)
- Working directory — the user's `pwd` (absolute path), **not** the repo root. The agent does not inherit `cwd` reliably and must run git from there without `cd`-ing to git toplevel.
- A note on the diff scope: "review the merged diff of working tree vs `$BASE` — this is the full task surface, regardless of which parts the user has committed yet"

The reviewer's evidence is the task contract and the diff. Do not pass session history or the rationale behind the changes — independence is the point.

### 2. Evaluate findings

Classify findings as critical, important, or a problem with the plan. For each:

1. Restate it. Ask the reviewer to clarify if its meaning is unclear.
2. Open the affected code and verify that the defect exists.
3. Check whether the proposed fix suits this design and codebase.
4. Decide to fix, reject with technical reasons, or defer to the user.

### 3. Announce the decision before editing

<required>
Before any fix goes in, tell the user what you decided for each finding. One line per finding:
- what the reviewer said,
- your verdict (fix / push back / defer),
- if fixing: the exact change you are about to make.

This is a short summary, not an approval pause: apply verified in-scope fixes straight after it. A review-only task stays read-only.
</required>

Example: "The parser accepts an empty identifier. Confirmed. Add rejection at the parser boundary and a regression test."

### 4. Apply fixes

Fix critical and important issues. Edit the file and `git add` the result. Do not commit.

Compare the staged result with the task file's seed line (the first paragraph under the title). Record a mismatch as an important finding and fix it in scope, without asking.

Run [uexecute](../uexecute/SKILL.md#consistency-pass)'s consistency pass for each changed rule or pattern so sibling paths do not remain inconsistent. If fixes are substantial, dispatch the reviewer again on the new diff.

### 5. Write the conclusion

```markdown
## Conclusion

Outcome: <one sentence on what was done — do not re-narrate the diff>

Invariants:
- IV1 — <verification evidence>

### Assumptions check
- AS1 — held | violated | unverifiable — <evidence or reason>

### Unknowns outcome
- UK1 — resolved | still-open — <answer or blocker>

Plan adherence: <deviations>

Review findings:
- Critical: <resolution>
- Important: <resolution or justified deferral>

Scope flag:
- <reviewer's flag, preserved for the user>

Future work:
- <item and design-scope justification or newly discovered fact>

Verified by: <material manual checks, deferred smokes, or other non-routine evidence>
```

Omit sections with no content. Do not omit violated assumptions or unresolved unknowns. A violated assumption needs evidence and, if it invalidates the outcome, a revised phase or a user decision. Preserve a scope flag for the user instead of silently redesigning the task.

## Receiving feedback — rules

<dispatcher-rules>
Never:
- "You're absolutely right" / "Great catch" / "Thanks for catching that"
- Implement blindly without verifying against the codebase
- Batch fixes without checking each independently
- Respond partial when multiple findings may be linked — clarify all first

Do:
- Verify against codebase reality before acting
- Push back with technical reasoning when the reviewer is wrong
- Ask for clarification when a finding is unclear
- Show the fix in a diff — actions over words

Pushback is legitimate when:
- The suggestion breaks existing behavior
- The reviewer lacks context only the design has (e.g. intentional tradeoff)
- The suggestion violates YAGNI (over-engineering an unused path)
- The suggestion conflicts with explicit design or invariant decisions
</dispatcher-rules>

## Never

- Accept "ready to merge" without evidence
- Hand back to the user with open Critical or Important findings (Kind `change`; Kind `review` hands every finding to the author)
- Skip the Conclusion write-up
- Run review on yourself (always use the subagent — preserve independence)
- Run `git commit`, `git commit --amend`, or `git push`. The user owns those

## Terminal state

Conclusion written, all Critical/Important resolved or explicitly deferred with justification → tell the user the workflow is done and remind them the staged-and-fixed changes are theirs to commit/amend. Set Status to `done` as the last write.
