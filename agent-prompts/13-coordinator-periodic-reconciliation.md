# Coordinator Periodic Reconciliation (Hermes cronjob Configuration)

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-21
> Reviewer: Richy (approved)

Goal: Hermes, as a cron scheduled task, performs one repeatable, idempotent ledger-consumption and health reconciliation for its iteration `<iteration-id>`, as the fallback for the event channel. Do not routinely iterate the task list or rely on cross-task API events to discover completion results.

## 1. cron Configuration

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
HermesInstanceRef: <hermes-instance-id>
LedgerLocation: <ledger location>
CanonicalTaskDocumentPath: <Git-managed task definition/snapshot path>
TaskDocumentBaselineRef: <immutable-ref>
CronInterval: <~1 minute; 5-field cron or equivalent>
StopConditions: <stop-conditions>
NotifyConditions: <notify-conditions>
```

cron precision is minute-level (per *Hermes Capability Boundary List* §4). Events (Feishu / Codex gateway polling) are consumed first; cron fallback reconciliation covers event loss.

## 2. Per-Round Order

1. Read the ledger, verify Schema, IterationID, StateRevision, and RecordID uniqueness.
2. On ledger missing, corrupted, or version rollback, enter `RecoveryOnly`, recover per `../specs/09-hermes-ledger-runtime.md` §13, forbid dispatch, and report recovery needs.
3. If `Paused=true`, do not consume, advance business state, create tasks, rework, Review, or integrate; output the pause state then end this round.
4. Collect all records with `SignalState=PendingConsumption`.
5. When there are no pending-consumption records, do not call the task list or read carrier-by-carrier; only check ledger integrity, document snapshot, and the Git ancestry relationships needed by the current gate.
6. When there are pending-consumption records, dedup by `RecordID + SignalRevision`, and read only the `ExecutionRef` bound by that record, not other execution carriers.
7. Verify Invocation, Task, role, branch, SHA, ReviewTarget/Verdict, or integration result; cross-verify with Git and immutable documents.
8. A verified result triggers one state transition and at most one new DispatchKey; then write `ConsumedAt/ConsumedBy`, the summary state, and the next-stage record, and increment `StateRevision`.
9. On targeted-read failure, retain `PendingConsumption`, record `ControlPlaneError`, and notify; do not switch to scanning other carriers or write it as code `Blocked`.

## 3. Prohibited

- Infer "no result" from UI idle/completed;
- Use cross-task API events or task-list summaries as the state-discovery entry;
- Scan all execution carriers when there is no `PendingConsumption`;
- Infer "Review has no conclusion" from unchanged Git HEAD;
- Treat a dispatch request identifier as failure;
- Re-dispatch tasks that already have a valid DispatchKey;
- Write ToolRuntime/read errors as code `Blocked`;
- Advance business state while `Paused` or `RecoveryOnly`.

## 4. This-Round Output

```text
ProtocolVersion:
EventType: ReconciliationCycleResult
IterationID:
HermesInstanceRef:
CoordinatorMode: Active / Paused / RecoveryOnly
CheckedAt:
TaskDocumentPath:
LedgerLocation:
StateRevisionBefore:
StateRevisionAfter:
ConsumedRevisionBefore:
ConsumedRevisionAfter:
PendingRecords:
TargetedExecutionsRead:
SignalsValidated:
SignalsConsumed:
GitHeadsChecked:
RealStateChanges:
NewDispatches:
ControlPlaneErrors:
BusinessBlockers:
NotificationDecision: Notify / Quiet
NextInspection:
```

`RealStateChanges: None` is only allowed after actually filling in document version, pending-consumption records, targeted execution carriers, and necessary Git checks. Notification rules prioritize reporting new conclusions, need for user authorization, ledger corruption, targeted-read failure, or real block; pure no-change is silent by default (notification 0 token, reads structured ledger fields + template assembly).

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex thread periodic-reconciliation prompt (shared JSON + short-lived lock + Watchdog counter) |
| V2.5 | 2026-08-20 | WorkBuddy | Rewritten as Hermes cronjob reconciliation config: ledger + event/cron dual channel; removed short-lived lock / Watchdog / shared JSON |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
| V2.5 errata | 2026-08-21 | WorkBuddy | Synced source errata bd6a71f: heading-level, wording, and reconciliation-terminology fixes |
