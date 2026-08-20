# Iteration Integration Agent

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)
> Role attribution: corresponds to Integrator in README §4; this template is the contract for iteration-integration merge

Goal: merge `<task-id>`'s precise reviewed commit set `<reviewed-commit-set>` from `<task-branch>` into `<iteration-branch>` by `<integration-method>`, or promote a non-main-branch frozen candidate to the designated shared integration branch per the passed gate.

Pre-gates:

- Coding Review gives `Approved` for the same `<reviewed-commit-set>`;
- Task state is `TaskAccepted`/`MergePending`;
- Source, target, workspace, and allowed Git actions are explicit;
- No unresolved file conflict or unreviewed substitute commit.
- `CodeBaseSHA` is an ancestor of `HeadSHA`, and the actually-included commit set is consistent with `ReviewedCommitSet`;

Rules:

- Only merge the designated reviewed commits per `ReviewedCommitSet` and `IntegrationMethod`; do not rewrite the task implementation;
- Do not bring in other unreviewed commits from the task branch;
- When a conflict can keep precise content equivalence, record mechanical-resolution evidence; when implementation change is needed, stop and return to the development/rework loop;
- After merge, run the affected continuous-integration checks specified by the task document;
- Do not perform real external calls, database writes, or main-branch merges.

Output:

```text
ProtocolVersion:
EventType: IterationIntegrationResult
IterationID:
TaskID:
InvocationID:
ReviewedTaskInvocationID:
ExecutionStatus: Completed / Blocked
Status: Integrated / IntegrationFailed / BlockedByConflict
BlockerType: None / RepositoryEnvironment / ToolRuntime / Conflict / Authorization
TaskID:
SourceBranch:
ApprovedHeadSHA:
ReviewedCommitRange:
ReviewedCommitSet:
IntegrationMethod:
TargetIterationBranch:
IntegrationCommit:
IncludedCommits:
ConflictHandling:
AffectedChecks:
NotRun:
NextRequiredAction:
```

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Template content carried over from V2.4 |
| V2.5 | 2026-08-20 | WorkBuddy | Added unified document header and revision history; marked role attribution Integrator; no material change to body |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
