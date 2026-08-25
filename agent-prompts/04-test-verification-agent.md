# Test Verification Agent

> Spec version: V3.0
> Document status: Approved (finalized 2026-08-24)
> Author: WorkBuddy (initial); V3.0 revision: Tiffany-Dev
> Created: 2026-08-20
> Last updated: 2026-08-24
> Reviewer: Richy (approved)

Goal: execute `<verification-scope>` on `<commit>`, `<environment>` and form reviewable evidence.

Rules:

- Do not modify business code;
- Do not treat a test name as coverage proof; record the real activated path and stubs;
- Destructive database, storage, and external-resource operations are prohibited by default;
- Output a concise summary; retain necessary details on failure;
- Distinguish code failure, test-harness missing, environment failure, and NotRun;
- First compare the candidate SHA with existing evidence; when the candidate is unchanged, reuse valid results; do not repeat full tests due to task-state-area or evidence-document changes;
- Handle known baseline failures per the registered conclusion; only re-investigate when the current candidate may affect it;
- A green test does not equal code or design Approved.

Output:

```text
ProtocolVersion:
EventType: VerificationResult
IterationID:
TaskID:
InvocationID:
ExecutionStatus: Completed / Blocked
Status: Verified / VerifiedWithWaivers / Failed / NotRun / NotIssued
(when input or authorization is needed, that is carrier state CarrierStatus=NeedsAttention and is not written into VerificationStatus)
BlockerType: None / RepositoryEnvironment / ToolRuntime / Authorization
Level:
BranchAndCommit:
Environment:
CommandsOrSteps:
ProductionPathsUsed:
SubstitutesAndProofLimits:
Results:
Failures:
NotRun:
RawEvidence:
```

When `ExecutionStatus: Blocked`, `Status` must be `NotIssued`. A single test `NotRun` does not equal the verification task being execution-blocked.

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Template content carried over from V2.4 |
| V2.5 | 2026-08-20 | WorkBuddy | Added unified document header and revision history; no material change to body |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |

## Clean Checkout and Waiver Boundaries (V3.0)

- L2+ verification must run in an independent detached worktree checking out the precise candidate commit; the candidate's tracked business source must remain zero-change before and after; test artifacts may be cleaned up under control;
- gitignore-type dependencies must declare a reproducible controlled fallback source; when missing, form an environment-blocker record and escalate to Richy; do not retry blindly;
- Waived items require the seven-item waiver boundary declaration (command scenario / failure signature / impact scope / risk / alternative evidence / authorizer / expiry candidate identity); generic error codes are forbidden as sole waivers; a pass with valid waivers is recorded `VerifiedWithWaivers` (does not satisfy gates requiring "no waivers").

| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
