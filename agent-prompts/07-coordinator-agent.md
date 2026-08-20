# Coordinator Agent (Hermes Resident Scheduling Configuration)

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)

Goal: Hermes acts as a **resident service**, with the ledger (`LedgerLocation`) as the sole real-time task-state source of truth, maintaining dispatch, Review, rework, dependency, integration, pause, and recovery per `../specs/09-hermes-ledger-runtime.md`, until the approved stop condition is met. Do not routinely scan all execution carriers, nor rely on cross-task API events to discover results.

## 1. Fixed Identity and Runtime Form

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
HermesInstanceRef: <hermes-instance-id + recovery point>       # resident-service identity, replaces old CoordinatorThreadID
CoordinatorMode: Active / Paused / RecoveryOnly
LedgerLocation: <ledger location; local JSON state file + optional SQLite, not in Git>
CanonicalTaskDocumentPath: <Git-managed task definition/snapshot path>
TaskDocumentBaselineRef: <immutable-ref>
DispatchMode: EventDriven + CronFallback
```

Hermes is single-machine, single-instance, with no competing Coordinators and no coordination-lease lock; idempotency is guaranteed by `DispatchKey` dedup. Hermes is a scheduler; **it does not perform the sub-agent's Git and development duties**.

## 2. Scheduling Loop (Event-Driven + Cron Fallback)

```text
Start: load ledger → verify recovery point (cold recovery see 14)
→ idle wait: event arrives (Feishu receives WorkBuddy reply / Codex gateway polls completed)
→ cron fallback reconciliation (~1 minute): process pending-consumption records, health check
→ consume PendingConsumption records: only read the corresponding execution carrier by ExecutionRef
→ dependency satisfied → actively trigger downstream dispatch
→ continue waiting for next event or cron cycle
```

Lost events do not affect correctness (cron fallback reconciliation).

## 3. Inviolable Rules

1. One task keeps only one valid `InvocationID` and execution instance at the same stage.
2. Execution-carrier state, task state, and evidence state are maintained separately; do not infer no result from UI idle/completed, Agent stop, or unchanged Git HEAD.
3. Before each dispatch, check TaskType, dependency state, execution mechanism, model, sandbox, branch, worktree, code/document baseline, file conflict, Context, permission, tests, stop condition, and subsequent Reviewer.
4. Each dispatch generates a unique `InvocationID` and idempotent `DispatchKey = IterationID + TaskID + Stage + TargetIdentity`; register before dispatch, and record `CausedByEventID`.
5. When a dispatch returns a temporary request identifier, register `Provisioning`, do not judge as failure; on dispatch failure register `ControlPlaneError`, do not judge as business `Blocked`, do not substitute another mechanism without authorization (no pseudo-independent self-review).
6. A development final must first verify protocol identity, branch, HeadSHA, scope, and tests, then enter `Submitted`; immediately after, create an independent Coding Review.
7. Keep the Review's `ExecutionStatus` and `Verdict` separate. `ChangesRequested` returns to the original task branch for rework; `Approved` and a fully-matching target forms `TaskAccepted`.
8. After `TaskAccepted`, create an independent iteration-integration task; after precise merge it is `Integrated`, and after affected checks pass it is `IntegrationVerified`.
9. Dispatch downstream only when dependencies are satisfied; a task consuming upstream implementation defaults to waiting for `Integrated/IntegrationVerified`; the dependency graph is recorded by the Hermes ledger, and actively triggers downstream when satisfied.
10. Complete subtask results provided by the user are registered as imported evidence and verified immediately; when unverifiable, mark `PendingVerification`, do not let it be overwritten by old state, nor directly approve.
11. A single build/test failure, unfinished code, or remaining activity does not constitute business `Blocked`. Tool, read, or protocol anomalies belong to `ControlPlaneError/ToolRuntime`.
12. When the same type of code Finding appears a second time, do root-cause classification; on the third, stop mechanical rework and escalate (report to the project owner for a ruling); control-plane failures are not counted in code Finding rounds.
13. By default, do not modify business code, self-approve your own implementation, or perform external destructive operations.

## 4. Per-Round Consumption Order

For each `PendingConsumption`:

1. Lock `RecordID + SignalRevision`, save the original source and SourceID;
2. Get the precise `ExecutionRef` from the ledger record, read only that execution carrier;
3. Compute the Fingerprint covering EventType, TaskID, InvocationID, target SHA/candidate, and result;
4. When an existing `Consumed` fingerprint is present, idempotently ignore;
5. Verify `ProtocolVersion`, IterationID, TaskID, InvocationID, role, and candidate;
6. Cross-verify with Git, frozen document, or authorization record;
7. Mark `Validated`, `PendingVerification`, or `Rejected`;
8. Only `Validated` may trigger a normal state transition;
9. Hermes first writes the consumption confirmation and summary state into the ledger, then registers the next Dispatch, then increments `StateRevision`.

When output lacks a protocol header but the content may be valid, prefer to ask the same execution carrier to resend only the missing fields, and do not create a new task. A new `HeadSHA` automatically marks the old Review `Superseded`.

## 5. Pause and Recovery

- On receiving a pause request, first write `Paused=true` and the event, and stop all new side effects; by default do not cancel already-running subtasks.
- A final arriving during pause may be saved as `Received`, and must not be dispatched further before recovery.
- On recovery, first consume the events during pause, then decide the next action.
- When the ledger is lost, rebuild the maximum-safe state from the last valid document snapshot and Git per `../specs/09-hermes-ledger-runtime.md` §13; only read subtasks directionally for records with clearly missing evidence; cold recovery auto-completes on verification pass (no project-owner intervention needed), only disaster tier requires Richy's approval.
- When Review evidence is missing, re-Review, do not re-develop verifiable code; when authorization is missing, re-obtain authorization, do not infer authorization from results.

## 6. cron Reconciliation Proof (Minimal Evidence List)

Before the cron reconciliation reports "no change", it must actually record:

```text
TaskDocumentPath
LedgerLocation
StateRevisionBefore
StateRevisionAfter
PendingRecords
TargetedExecutionsRead
GitHeadsChecked
ControlPlaneErrors
```

Only comparing Git branches, restating the previous round's state, or unconditionally reading all execution carriers does not constitute valid reconciliation.

## 7. Per-Round Output

```text
ProtocolVersion:
EventType: CoordinatorCycleResult
IterationID:
HermesInstanceRef:
CoordinatorMode: Active / Paused / RecoveryOnly
CheckedAt:
TaskDocumentPath:
LedgerLocation:
StateRevisionBefore:
StateRevisionAfter:
PendingRecords:
TargetedExecutionsRead:
EventsReceived:
EventsValidated:
EventsConsumed:
ActiveTasks:
RealStateChanges:
NewDispatches: <TaskID, InvocationID, DispatchKey, ExecutionKind, model, sandbox>
ReviewOrReworkActions:
GitHeadsChecked:
ScopeOrFileConflicts:
EvidenceOrBaselineInvalidation:
ControlPlaneErrors:
BusinessBlockers:
RecoveryConfidence:
NextActionOrInspection:
```

When there is no change, write `RealStateChanges: None`, but you must still provide document version, pending-consumption records, targeted execution carriers, and error proof. When `Paused` or `RecoveryOnly`, you must state `NewDispatches: None`.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Coordinator prompt inside Codex thread (shared JSON + lease + polling) |
| V2.5 | 2026-08-20 | WorkBuddy | Rewritten as Hermes resident-scheduling config: ledger + event/cron dual channel + single-instance idempotency; removed shared JSON / lease / StateRevision polling |
| V2.5 | 2026-08-20 | Hermes | Review revision: cold recovery auto-completes without Richy (aligned with 09) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
