# Review: <ID> <title>

Branch: `<branch>` against `<baseBranch>`

## Untracked before start
<output of git ls-files --others --exclude-standard, or "none">

## Diff
<git diff --stat output>

## Checks
| Check | Result | Notes |
| --- | --- | --- |
| <command> | pass / fail / not run | <real output, trimmed> |

## AI review
<findings, and which were fixed; "not run" if aiReview is false>

## Acceptance
- [ ] <criterion> - evidence: <what was run or seen>, or "no evidence"

## Human checklist
Open and try, in order:
1. <concrete step>
2. <concrete step>

## Known gaps and risks
- <gap>
