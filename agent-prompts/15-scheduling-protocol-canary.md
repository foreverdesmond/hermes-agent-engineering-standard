# Scheduling-Protocol Canary (Hermes Closed-Loop Canary Configuration)

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)

Goal: without modifying business code or connecting to real external resources, verify `<iteration-id>`'s task dispatch, ledger state production/consumption, idempotency, pause, and recovery protocol. When Canary fails, real high-risk tasks must not be launched; an existing iteration may only have a temporary waiver explicitly recorded by the project owner.

## 1. Input

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
CanaryInvocationID: <invocation-id>
HermesInstanceRef: <hermes-instance-id>
LedgerLocation: <isolated-canary-ledger>
CanonicalTaskDocumentPath: <isolated-shared-document-path>
DispatchMode: EventDriven + CronFallback
ExpectedExecutionKind: <execution-kind>
ImplementerModel: <model>
ReviewerModel: <model>
SafeRepositoryTarget: <read-only or disposable target>
```

## 2. Rules

- Use an isolated Canary ledger, task-document snapshot, derived-evidence directory, and a no-business-side-effect target;
- Do not modify production code, business branches, databases, external services, or real config;
- For each check, retain the original dispatch result, ExecutionRef, StateRevision, state record, targeted final, Fingerprint, and state-transition evidence;
- On failure, stop the real iteration launch; do not cover up problems by expanding permissions.

## 3. Minimal Scenarios (retains the spirit of the original 14 scenarios + new resident scenarios)

1. Dispatch returns only a temporary request identifier, enters Provisioning and finally binds a formal `ExecutionRef`;
2. The execution carrier completes before the cron reconciliation cycle, first reports `PendingConsumption`, and Hermes consumes immediately per the ledger version;
3. Push/polling event sources are unavailable, but the ledger state record exists, and the corresponding final can still be precisely read;
4. On missing protocol fields, the same execution carrier resends, without creating duplicates;
5. Re-reading the same `RecordID + SignalRevision` is processed only once;
6. Import a user-provided structured result, entering verification rather than directly negating / approving;
7. Simulate a ToolRuntime error, not polluting business Blocked;
8. A new Head makes the old Review Superseded;
9. After Paused, no new dispatch or state side effect;
10. Hot recovery depends only on the ledger and necessary Git facts, preserving state and idempotency keys;
11. After deleting the derived Canary ledger, the state source of truth can still continue scheduling;
12. When the designated execution carrier fails, it does not trigger another execution mechanism as fallback (unless `ApprovedEquivalent` authorized);
13. Immutable document baselines read from different worktrees are consistent, and runtime state is written into the same ledger through Hermes;
14. Sandbox tiers are correct: workspace-write blocks git, danger-full-access allows git;
15. **[new] Multi-instance de-dup**: simulate two trigger sources, at most one produces a side effect, and DispatchKey idempotent dedup takes effect;
16. **[new] cron fallback after event loss**: simulate event loss, cron reconciliation can still consume without re-dispatching;
17. **[new] Hermes restart hot recovery**: simulate process restart, state is consistent after reading ledger + Git rebuild;
18. **[new] Canary-failure handling loop**: simulate a scenario failure entering `CanaryFailed`, record failed scenario / root cause, responsible-party fix, re-run, alert Richy on 3 consecutive failures, and only after pass lift the real task's `Planned/Ready` freeze (consistent with `../specs/09-hermes-ledger-runtime.md` §14.1).

## 4. Output

```text
ProtocolVersion:
EventType: CoordinatorCanaryResult
IterationID:
CanaryInvocationID:
ExecutionStatus: Completed / Blocked
ScenarioResults: <1-18 Pass/Fail + evidence>
DuplicateDispatchCount:
MissedEventCount:
IncorrectStateTransitionCount:
UnauthorizedFallbackCount:
RecoveryComparison:
ControlPlaneErrors:
CanaryFailedScenarios: <failed scenario number + root cause + handling progress + whether Richy alerted>
Verdict: CanaryPassed / CanaryFailed / NotIssued
RequiredFixes:
```

`CanaryPassed` is allowed only when all 18 items pass, the four error counters are all 0, and the recovery comparison is consistent. When any scenario enters `CanaryFailed`, handle per `../specs/09-hermes-ledger-runtime.md` §14.1: record root cause → fix → re-run → alert Richy on 3 consecutive failures; real high-risk iterations stay frozen until Canary passes or Richy explicitly records a temporary waiver.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex scheduling-protocol Canary prompt (isolated shared JSON + lock path + Codex-creation fallback ban) |
| V2.5 | 2026-08-20 | WorkBuddy | Rewritten as Hermes closed-loop Canary: removed isolated shared JSON / lock / Codex fallback ban; added multi-instance de-dup, cron fallback after event loss, Hermes restart hot recovery |
| V2.5 (pending review) | 2026-08-20 | Hermes | Added scenario 18 Canary-failure handling loop (CanaryFailed → fix → re-run → alert Richy on 3 consecutive failures), output added CanaryFailedScenarios field, consistent with 09 §14.1 |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
