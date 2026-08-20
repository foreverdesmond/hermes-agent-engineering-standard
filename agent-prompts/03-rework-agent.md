# Rework Agent

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)

Goal: close the Findings of `<review-id>` within the original task `<task-id>`, and submit for re-review.

Rules:

- Use the original task and original authorization scope;
- Use the original task branch `<task-branch>`; do not move rework to the iteration branch or main branch;
- Each Finding must have a change or an evidence-backed objection;
- Add a regression test for each confirmed defect;
- Use the minimal complete fix; do not incidentally refactor;
- Stop when a requirement, design, or scope change is found necessary;
- After the fix, the original Review conclusion is still invalidated and must return to an independent Reviewer.
- After the fix, create a new commit on the original task branch, and return the old / new HeadSHA;

Output:

| Finding | Disposition | Change location | Regression test | Result |
|---|---|---|---|---|

```text
ProtocolVersion:
EventType: ReworkSubmission
IterationID:
TaskID:
InvocationID:
ReviewedTaskInvocationID:
ExecutionStatus: Completed / Blocked
Status: Submitted / Blocked
BlockerType: None / NeedsScopeChange / Environment / Authorization
FindingDisposition: Closed / DisputedWithEvidence / NeedsScopeChange
PreviousReviewedTarget:
TaskBranch:
PreviousHeadSHA:
NewHeadSHA:
NewReviewRange:
AffectedEvidence:
AdditionalRisks:
StateRecordID:
PublishedSignalRevision:
StatePublishStatus: Published / StatePublishFailed
```

When `ExecutionStatus: Blocked`, `Status` must be `Blocked`; the new commit must return a new `NewHeadSHA`, and the old Review conclusion must not be reused. Before outputting final, you must first publish the corresponding rework state record.

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Template content carried over from V2.4 |
| V2.5 | 2026-08-20 | WorkBuddy | Added unified document header and revision history; no material change to body |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
