# Verification Skill Patterns

Read this file when the skill being designed exists to check work: it verifies
code, output, or a running product, and fixes or reports what fails.

A verification skill closes a loop: check the work, fix what fails, check again,
and only then report done. Structurally it is a hybrid skill. What sets it apart
is the content below.

## Start From a Real Manual Check

Ask the user which check they repeat by hand or which correction they keep
making. Write that procedure in plain English, as for a new teammate, before
structuring anything. If a built-in already covers it (`/verify`, the linter,
the test suite), use that and do not write a skill.

## Decide Where the Check Runs

| Placement | Use when | Cost |
|---|---|---|
| **Standalone** | Cross-cutting check run deliberately: pre-commit scan, accessibility audit | Someone has to remember to run it |
| **Embedded** | The check belongs to one producing skill the user controls; it becomes that skill's final step | Cannot be added to built-in or plugin-managed skills, which are overwritten on update |
| **Chained** | The producing skill cannot be edited; a wrapper skill runs it, then invokes the check | More tokens per run; less flexible |
| **On every PR** | The chain has stopped changing and the team wants it enforced | Premature while the checks are still being tuned |

If the user would run a standalone check after every change, embed or chain it.

## Write Checks That Can Fail

| Kind of check | Write it as |
|---|---|
| Rule over code or a diff | A deterministic statement: "Reject any migration that drops a column without a backfill step." |
| Behavior of a running product | A driver script in `scripts/` (browser automation, a terminal driver) with an assertion on state after each step |
| Output against a spec | The spec file's path and the fields to compare |

Put the driving and asserting in scripts. Claude then spends its turns on
interpreting failures, not on rebuilding the harness.

## Define "Passed"

State, in the skill, what must be true for the check to pass and what Claude
shows as proof: the command output, a screenshot, a recording. A check that
ends in "looks good" has not passed.

## Close the Loop

- Report each failure with `file:line` and what was expected.
- Fix it, then run the check again. Do not report completion on a failing check.
- Set a stop condition: after a fixed number of failed attempts, stop and
  report what is still failing. Otherwise the loop can run indefinitely.

## Gotchas

- A check that cannot fail verifies nothing. Before trusting it, break the code
  on purpose and confirm the check catches it.
- A success status is not proof of the outcome. Assert on the resulting state,
  not on the absence of an error.
