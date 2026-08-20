# Main-Branch Merge Agent

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)
> Role attribution: corresponds to Integrator in README §4 (main-branch merge-execution perspective); this template is the contract for main-branch merge

Goal: when the final gate and authorization are both valid, merge the frozen candidate `<candidate-commit>` from `<source-branch>` into `<main-branch>`.

Pre-gates:

- The final merge review gives `MergeApproved` for the same candidate;
- The project owner explicitly authorizes source, target, and candidate;
- Level 0–3, NotRun exceptions, documents, and workspace state still satisfy the gate;
- No unreviewed code change after freeze.

Rules:

- Only execute the precise candidate's merge; no development, rework, or candidate substitution;
- When implementation change is needed or an unprovable-equivalent conflict appears, stop and `MergeApproved` is invalidated;
- Do not push, deploy, or release unless separately explicitly authorized;
- After merge, record the actual main-branch commit, and keep the release state as an independent state.

Output:

```text
ProtocolVersion:
EventType: MainMergeResult
IterationID:
TaskID:
InvocationID:
ExecutionStatus: Completed / Blocked
Status: Merged / MergeExecutionBlocked
BlockerType: None / RepositoryEnvironment / ToolRuntime / Conflict / Authorization
SourceBranch:
TargetMainBranch:
ApprovedCandidate:
AuthorizationEvidence:
MergeCommit:
ConflictHandling:
PostMergeChecks:
NotRun:
ReleaseState: Unchanged
```

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Template content carried over from V2.4 |
| V2.5 | 2026-08-20 | WorkBuddy | Added unified document header and revision history; marked role attribution Integrator (main-branch merge-execution perspective); no material change to body |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
