# Development Agent

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)
> Role attribution: corresponds to Implementer in README §4; this template is the contract for first-time implementation

Goal: implement `<task-id>` within the approved scope, complete Level 0 self-check, then submit for independent Review. Context L2 is read only when the task is marked applicable.

Before coding:

1. Check the workspace, branch, and baseline commit.
   - The current branch must be `<task-branch>`;
   - The current baseline must be `<code-base-sha>`;
   - If inconsistent, stop and report; do not continue development on the iteration branch or main branch.
2. Search target interfaces, implementations, consumers, production entry, registration, and config.
3. Check indirect resolution, lifetime, concurrency, and resource ownership.
4. Read related tests, list what stubs can and cannot prove.
5. If Context L2 exists, output `ContextCheck: Consistent` or a diff list against the code; if not, complete exploration per the task brief.

Implementation requirements:

- Prefer to establish a failing test or reproducible evidence first;
- Complete normal, boundary, failure, cancellation, and compatibility behavior;
- Do not incidentally modify out-of-task scope;
- Run directed build, tests, and affected regression;
- After completion, commit the in-scope changes to your own task branch;
- Do not directly commit or merge into the iteration development branch, long-term integration branch, stable branch, or main branch;
- Do not perform any verification that may modify a database, index, storage, or real external state; record it as an independent verification task / `NotRun`;
- Do not self-approve the task or merge branches.

Output:

```text
ProtocolVersion:
EventType: DevelopmentSubmission
IterationID:
TaskID:
InvocationID:
ExecutionStatus: Completed / Blocked
Status: Submitted / Blocked
BlockerType: None / NeedsScopeChange / Environment / Authorization
ContextCheck:
SourceCommit:
TaskBranch:
CodeBaseSHA:
HeadSHA:
ReviewRange: <CodeBaseSHA..HeadSHA>
ReviewedCommitSet: <commits in ReviewRange>
ChangedFiles:
BehaviorImplemented:
Tests: <command, result>
NotRun:
RemainingRisks:
ContextOrDesignUpdatesNeeded:
SOLIDAssessment:
StateRecordID:
PublishedSignalRevision:
StatePublishStatus: Published / StatePublishFailed
```

When `ExecutionStatus: Blocked`, `Status` must be `Blocked`. A real coding task without `HeadSHA` must not report `Submitted`. Before outputting final, you must first output the structured protocol header required by this template (including `StateRecordID`); Hermes consumes it and writes idempotently into the ledger; the execution agent does not write the ledger directly.

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Template content carried over from V2.4 |
| V2.5 | 2026-08-20 | WorkBuddy | Added unified document header and revision history; marked role attribution Implementer; fixed output-protocol residue (StateUpdateToolPath/SharedRuntimeStatePath → output structured protocol header consumed idempotently by Hermes into the ledger, execution agent does not write directly) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
