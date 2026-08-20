# Coordinator Cold Recovery (Hermes Restart Recovery Rules)

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)

Goal: when the ledger is missing, corrupted, version-rolled-back, or a recovery point cannot be proven, rebuild `<iteration-id>`'s maximum-safe state from the last valid document snapshot and Git. Only directionally read the corresponding execution carrier when the document record points to pending-consumption or missing evidence. During recovery, do not advance business state or dispatch follow-up tasks.

## 1. Recovery Tiers (per *Hermes Capability Boundary List* §7)

| Tier | Trigger | Hermes action | Failure handling |
|---|---|---|---|
| Hot recovery | One-off tool/call occasional anomaly | Log, auto-resume from most recent ledger snapshot; retry | Still failing, wait for cron reconciliation fallback |
| Cold recovery | Process crash/restart | Fully auto-read ledger + Git rebuild `ConsumedRevision` → `RecoveryOnly` verification → recover | **Failure auto-escalates to disaster tier** |
| Disaster recovery | Ledger lost/corrupted (or cold recovery fails) | Rebuild from Git task-document snapshot, determine provable state, downgrade the rest to pending-verification | **Requires project-owner (Richy) intervention and approval** |

## 2. Cold-Recovery Input

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
RecoveryInvocationID: <invocation-id>
HermesInstanceRef: <hermes-instance-id>
LedgerLocation: <ledger location>
CanonicalTaskDocumentPath: <Git-managed task definition/snapshot path>
TaskDocumentBaselineRef: <immutable-ref>
Repository: <repository>
IterationBranch: <iteration-branch>
TaskBranchPattern: <pattern>
KnownTaskIDs: <list>
```

## 3. Forced Mode

```text
CoordinatorMode: RecoveryOnly
DispatchAllowed: false
IntegrationAllowed: false
BusinessWritesAllowed: false
```

## 4. Execution Steps

1. Read the immutable task definition; prefer to verify the ledger; when it is missing or corrupted, recover the task graph, roles, dependencies, baselines, stable state, and unconsumed records from the last structurally-valid `TASK-STATE-EXCHANGE` snapshot in `CanonicalTaskDocumentPath`.
2. Scan task branches, iteration branches, worktrees, commits, ancestry, and branch inclusion.
3. Only for stages recorded as `PendingConsumption`, `PendingVerification`, or missing evidence, directionally find the execution carrier by the `ExecutionRef` in them.
4. Only read the above target task history; forbid recovery as a full task-list / carrier scan.
5. Compute a Fingerprint for each piece of evidence, recording source, original time, recovery time, and confidence.
6. Verify by responsibility: document proves the plan, Git proves the code, Reviewer final proves the Verdict, execution-carrier history proves execution, explicit message proves authorization.
7. Compute the maximum-safe state for each task; do not infer Review or authorization from code results.
8. Mark duplicate instances, wrong execution mechanisms, invalidated Reviews, unknown merges, missing evidence, and conflicts.
9. Write back to the ledger and save the old-state backup; after stabilization update the development-task document snapshot; the derived evidence directory stores only long evidence.
10. Output the recovery report; cold recovery **auto-completes on verification pass (no project-owner intervention needed)**, and exits `RecoveryOnly` resuming scheduling on verification pass; only when cold recovery fails and escalates to disaster tier does it need project-owner (Richy) intervention and approval.

## 5. Maximum-Safe-State Rules

- Commit only: `CodePresence=PresentInTaskBranch`, `TaskState=InProgress` (existing state);
- Development final + Git match: at most `Submitted`;
- Valid `ChangesRequested`: recover that state and look for a new Head;
- `Approved` precisely matches current Head: recover `TaskState=TaskAccepted`, but do not dispatch on this before recovery audit passes;
- Current Head changed: old Review `Superseded`;
- Code merged but no gate evidence: `IntegrationUnverified`;
- Review history lost: recommend re-Review, not re-development;
- Authorization lost: re-obtain authorization.

## 6. Output

```text
ProtocolVersion:
EventType: RecoveryReport
IterationID:
RecoveryInvocationID:
HermesInstanceRef:
CoordinatorMode: RecoveryOnly
TaskDocumentBaselineRef:
GitRefsScanned:
ExecutionsDiscovered:
ExecutionHistoriesRead:
RecoveredEvents:
RecoveredTaskStates:
SupersededEvidence:
DuplicateOrInvalidInvocations:
IntegrationUnverified:
MissingEvidence:
Conflicts:
RecoveryConfidenceByTask:
LedgerWrittenAt:
RecoveredStateRevision:
TargetedExecutionsRead:
RequiredReReviews:
RequiredOwnerDecisions:
Verdict: Ready / RecoveryIncomplete
```

Cold recovery can switch to Active after auto-verification passes; even with `Verdict: Ready`, when cold recovery fails and escalates to disaster tier and project-owner approval is not yet obtained, Hermes must not dispatch tasks on its own.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex thread cold-recovery prompt (shared JSON rebuild + short-lived lock writeback) |
| V2.5 | 2026-08-20 | WorkBuddy | Rewritten as Hermes restart recovery rules: three-tier recovery (hot/cold/disaster); removed shared JSON rebuild / short-lived lock |
| V2.5 | 2026-08-20 | Hermes | Review revision: cold recovery auto-completes on verification pass without Richy (only disaster tier needs); EvidenceIncomplete → InProgress |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
