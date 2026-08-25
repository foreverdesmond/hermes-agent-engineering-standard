# Hermes Ledger & Runtime Spec

> Spec version: V3.0
> Document status: Approved (V3.0 final baseline)
> Applicability: development iterations led by the resident Coordinator (Hermes), with WorkBuddy / Codex / Human collaborating in parallel
> Predecessor: `09-Scheduling Control Plane & Runtime Ledger Spec` (V2.4, Codex thread as Coordinator)
> Revised: 2026-08-24
> Foundation: *Hermes Capability Boundary List* (measured capability), *Hermes Process & Boundary Resolution* (boundary decisions)
> Author: Tiffany-Dev (V3.0 revision; initial rewrite = WorkBuddy)
> Reviewer: Richy (approved)

## 1. Purpose and Single Responsibility

This document specifies the control plane under the resident Coordinator (Hermes) mode, including:

- The Hermes ledger (local JSON state file + optional SQLite) as the sole source of truth for task runtime state;
- Dispatch, execution-carrier identity binding, state production/consumption, and idempotency;
- The dual-channel scheduling loop of event-driven + cron fallback reconciliation;
- Separation of three state types (execution-carrier state, task state, evidence state);
- Idempotent dedup under a single instance (no coordination-lease lock);
- Pause, three-tier recovery (hot/cold/disaster), and maximum-safe state;
- Dispatch-sandbox tiers and the `danger-full-access` authorization boundary;
- Canary acceptance for the scheduling protocol.

Role selection and prompt assembly follow `07-subagent-delegation-prompts.md`; development, Review, verification, and merge gates follow `08-verification-review-merge-gate.md`. This document defines the control protocol; the Hermes ledger is the sole real-time source of truth for that iteration's task runtime state, and the `TASK-STATE-EXCHANGE` block in `CanonicalTaskDocumentPath` holds the task definition and the most recent persistent snapshot.

This spec does not authorize business-code modification, external operations, Review approval, or branch merge. Hermes is a scheduler; **it does not perform the sub-agent's Git and development duties**.

## 2. Core Principles

1. **Document-state-driven**: without a higher-version pending-consumption state and unambiguous target in the development-task document, do not start cross-task reading or advance task state.
2. **State separation**: whether the execution carrier is running, what stage the code task has reached, and whether evidence is consumed must be maintained separately.
3. **Durable and recoverable**: the scheduler does not depend on chat context, UI labels, or model memory; on Hermes restart it can read the ledger + Git to rebuild.
4. **Document-first publish, at-least-once consumption**: the subtask first reports results; re-reading the same record is allowed, but producing a side effect twice is not.
5. **Maximum-safe state**: on recovery, restore only to the highest state the evidence supports; unprovable content stays pending-verification.
6. **Control-plane failure does not pollute business state**: dispatch, event-read, or ledger-write failures must not be disguised as code `Blocked` or a Review Finding.
7. **Single-owner idempotency**: Hermes is single-machine, and each iteration has exactly one current scheduling owner identified by the monotonically increasing `CoordinatorEpoch`; idempotency is guaranteed by `DispatchKey` dedup and stale-Epoch fencing, with no coordination-lease lock.
8. **Event-driven + scheduled fallback**: event sources take priority, scheduled reconciliation is the fallback; lost events do not affect correctness.
9. **Explicit execution-mechanism configuration**: each dispatch records `ExpectedExecutionKind`, model, and sandbox; the execution carrier may not be substituted without authorization.

## 3. Three State Types

### 3.1 Execution-Carrier State (formerly ThreadStatus)

Describes the running state of the execution carrier, not the development-task conclusion:

| State | Meaning |
|---|---|
| NotCreated | Not yet dispatched/created |
| Provisioning | Dispatch sent, formal execution identifier not yet bound |
| Running | Running or still resumable |
| Idle | No current activity; a consumable final may already exist |
| NeedsAttention | Requests input, authorization, or a recoverable exception occurs |
| Completed | Execution ended; the final must be read on a pending-consumption signal before judging the task conclusion |
| Unavailable | Currently unreadable or the execution carrier is lost |
| Cancelled | The execution carrier has been explicitly cancelled |

When the adapter returns a formal execution identity, register it directly; if only a provisional identifier is returned, that means `Provisioning`, and must not be judged as creation failure. Use `ExecutionRef` uniformly to reference the execution carrier, without distinguishing provisional/formal identity stages.

### 3.2 Task State (TaskState)

Describes the development-flow state:

```text
Planned → ContextGenerationPending → Ready → InProgress → Submitted → InReview
  → ChangesRequested → ReworkInProgress → Submitted
  → TaskAccepted → MergePending → Integrated → IntegrationVerified
```

`ContextGenerationPending` appears only when `ContextL2Policy=Required`: Hermes first dispatches the Doc/Design Reviewer to generate Context L2 (after its dependencies are satisfied); only after its completion signal is written back to the ledger is the task advanced to `Ready`, then the Implementer is dispatched.

`Blocked` and `Cancelled` are evidence-backed bypass states. Candidate versions continue to use `ComponentVerified`, `SystemVerified`, `ExternalVerified`, `MergeApproved`, and `Merged`.

### 3.3 Evidence State (EvidenceState)

Describes whether an event or conclusion has been reliably consumed:

| State | Meaning |
|---|---|
| Unread | Known possibly to exist but not yet read |
| Received | Content obtained, verification not yet complete |
| PendingVerification | Source exists but cannot be cross-verified temporarily |
| Validated | Identity, target, and content have been verified |
| Rejected | Evidence does not belong to the target task, format is unrecoverable, or content is invalid |
| Superseded | Replaced by a new `HeadSHA`, new candidate, or approved alternative evidence |
| Consumed | An idempotent state transition or subsequent action has completed |

The following inferences are strictly forbidden:

- `Idle`, `Completed`, or UI stop ⇒ no Review conclusion;
- Git HEAD unchanged ⇒ no Review, help request, or environment event;
- A dispatch request identifier not yet bound to a formal execution identifier ⇒ dispatch failure;
- Test `NotRun` ⇒ Review necessarily blocked;
- Code already in the iteration branch ⇒ compliant with `Integrated`.

## 4. Immutable Execution Baselines

Each dispatch must bind the following baselines; when not applicable, state `N/A` and the reason:

```text
CodeBaseSHA
RequirementsBaselineRef
DesignBaselineRef
TaskDocumentBaselineRef
ContextBaselineRef
```

Real coding tasks default to using Git identity:

```text
TaskBranch
WorktreePath
CodeBaseSHA
HeadSHA
ReviewedCommitRange
ReviewedCommitSet
MergeTarget
```

Document baselines must use immutable Git commit/tree, controlled-document revision, diff hash, or a project-owner-approved stable snapshot. Do not treat the absolute path of another continuously-changing workspace as the sole design baseline.

## 5. Ledger State Surface

### 5.1 Sole Source of Truth for State

- **The Hermes ledger** (`LedgerLocation`, local JSON state file + optional SQLite, **inside the project directory, not committed to Git**) is the **sole real-time source of truth for task runtime state** during the active iteration, maintained by the current CoordinatorEpoch owner. **The ledger must never be committed to Git—even if the execution carrier has `danger-full-access` commit permission, no add/commit may include ledger files** (the ledger changes with development and would create a circular reference / self-contained hash with commits, and must be isolated from the version repository).
- **The development-task document** (`CanonicalTaskDocumentPath`) `TASK-STATE-EXCHANGE` block is the **Git persistent snapshot**, used for cross-restart / disaster recovery.
- The execution Agent **does not write the ledger directly**; it reports structured results through the carrier channel, which Hermes consumes and writes into the ledger (see §8).
- The ledger stores only summaries, SHAs, Verdict, `ExecutionRef`, and next-action needed for locating and verifying; long logs, full diff, and full final are not written into the ledger, but fall into the derived evidence directory.

### 5.2 Iteration-Level Minimal Fields

```text
SchemaVersion, ProtocolVersion, IterationID, HermesInstanceRef,
CoordinatorMode, Paused, CodeBaseSHA, RequirementsBaselineRef,
DesignBaselineRef, TaskDocumentBaselineRef, CanonicalTaskDocumentPath,
LedgerLocation, StateRevision, ConsumedRevision, UpdatedAt,
CoordinatorEpoch{Epoch, Owner, StateRevisionAtTakeover, LostOwnerTimeout},
PolicyVersion, PolicyArtifactDigest
```

V3.0 new required fields: `CoordinatorEpoch` (top-level object; §12.3 sole source of truth for scheduling authority), `PolicyVersion`, and `PolicyArtifactDigest` (currently effective carrier policy artifact, §12.4).

### 5.3 Task-Level Minimal Fields

```text
RecordID, TaskID, TaskType, Stage, InvocationID, ParentIterationID, Role,
ExpectedExecutionKind, ExpectedModel, ActualModel, ModelProvider, Sandbox,
TaskBranch, WorktreePath, CodeBaseSHA, HeadSHA,
ExecutionRef, CarrierStatus, TaskState, EvidenceState,
SignalRevision, SignalState, ProducedAt, ConsumedAt, ConsumedBy,
LastEventFingerprint, BlockerType, RecoveryConfidence, NextAction,
MaxAutomaticAttempts, AutomaticAttemptsCount, ExecutionFailureType,
PolicyVersion, PolicyArtifactDigest, DispatchedCoordinatorEpoch
```

V3.0 new required fields:

- **Task/stage level**: `PolicyVersion` + `PolicyArtifactDigest`—the policy may change within one iteration; only the dispatch-time policy snapshot can prove "why that carrier was chosen for that dispatch"; `DispatchedCoordinatorEpoch`—the scheduling-authority epoch at dispatch time (iteration level keeps only the current owner state);
- `MaxAutomaticAttempts` (default 3, cap 3), `AutomaticAttemptsCount` (persisted count, not reset by owner change or instance replacement), and `ExecutionFailureType` (execution-failure classification, §8).

`ExpectedModel` is what the project requires; when `ActualModel` cannot be verified, record `Unknown`, and do not claim a model match on your own.

## 6. Events, Idempotency, and Dispatch Dedup

### 6.1 Event Sources and Dual-Channel Scheduling

Event sources are defined by **functional type**; each type's concrete channel implementation is a deployment fact, registered in the instance capability record:

| Functional event source | Coverage | Nature |
|---|---|---|
| Push event source | Receive interactive-carrier reply messages | Event, may be lost |
| Polling event source | Query async-carrier execution progress | Event, active polling |
| Integration writeback | After the Integrator merges the precise commit and runs affected integration checks, it reports a structured protocol header (with `IntegrationCommit` + `IntegrationStatus`) through its carrier channel, which Hermes consumes into the ledger | Event, reported by the execution carrier, not depending on an external webhook |
| Candidate freeze (human-gated) | After the project owner freezes the iteration candidate and informs Hermes (or Hermes polls the freeze marker), it triggers dispatching Level 1 | Event, human-gated |
| Scheduled reconciliation fallback | Reconciliation loop backfilling push-event loss | Fallback, covers event loss |
| Key-node notification output | Notify the project owner of key nodes | Output |

**Conclusion**: event sources are consumed first, scheduled reconciliation is the fallback; lost events do not affect correctness.

> **IntegrationVerified writeback note**: after `Integrated`, `IntegrationVerified` no longer depends on an external CI webhook. The execution subject is the **Integrator** (or an independent `IntegrationValidationTask`): after it merges the precise commit, it runs/triggers affected integration checks and reports a structured protocol header (with `IntegrationCommit` + `IntegrationStatus=Passed/Failed`) through its own carrier channel; Hermes consumes this signal into the ledger and triggers downstream Level 1. A missing report is treated as a pending-consumption signal, escalated by cron stall detection (see §11).

### 6.2 Idempotent Identity

- Dispatch idempotency key: `DispatchKey = IterationID + TaskID + Stage + TargetIdentity`. When the same `DispatchKey` already has a valid instance, re-dispatch is forbidden.
- Consumption idempotency key: `RecordID + SignalRevision`. Re-reading the same record only updates read metadata, and does not repeatedly create Review, rework, or integration tasks.
- **RecordID global uniqueness (V3.0)**: `RecordID` must be generated as a non-reusable ULID/UUID; manual sequence reuse is forbidden; before generation, de-duplicate across all Tasks and Signals. Conflict handling: freeze the conflicting record, generate a new ID, and retain an old-to-new mapping audit. The pre-dispatch gate enforces fail-closed validation of RecordID uniqueness.
- Event fingerprint `Fingerprint` covers at least event type, task, Invocation, target SHA/candidate, and normalized result.

### 6.3 Causality

Any new dispatch that produces a side effect must record:

```text
CausedByEventID
DispatchKey = IterationID + TaskID + Stage + TargetIdentity
```

## 7. Dispatch and Identity Binding

### 7.1 Dispatch Adapter Contract (V3.0: abstract contract; implementation details belong to the instance registry)

The scheduler interacts with execution carriers through **dispatch adapters**. Available carriers are decided by the current version of the "carrier policy artifact" (§12.4); this spec enumerates no concrete carriers and prescribes no endpoints, parameters, or vendor configuration.

Contract every dispatch adapter must satisfy:

| Contract item | Requirement |
|---|---|
| Dispatch | Accept a structured dispatch intent and submit it to the target carrier |
| Identity binding | Return a traceable execution identity (ExecutionRef) for subsequent directed reads |
| Sync/async results | Support synchronous return or an asynchronous completion signal; when asynchronous, a consumable completion event must eventually be produced |
| Failure semantics | Unavailability/failure must be reported explicitly (ControlPlaneError); silent drops are forbidden |

> Concrete endpoints, request parameters, vendors, and channel configuration are deployment facts, registered in the **instance capability registry** (a non-normative controlled record), updated as the deployment evolves, and never entering the spec text.

### 7.2 ExecutionRef Unified Identity (V3.0: platform-neutral)

- All execution carriers are uniformly modeled as `ExecutionRef`—a traceable execution identity returned by the adapter; no platform-specific ID model or format is prescribed.
- **Provisional and formal identity**: an adapter may first return a **provisional identity** (register `Provisioning` at that point); once the formal identity is ready it **replaces** the provisional one; the replacement must leave an audit trail and must not produce two coexisting formal identities.
- When a dispatch-create call fails, register `ControlPlaneError`, do not register business `Blocked`, and do not fall back to another execution mechanism (unless the project owner approves `ApprovedEquivalent`).
- **Carrier-unavailable escalation threshold**: if the same execution carrier (identified by `ExecutionRef`) fails dispatch/read consecutively **3 times** and remains unavailable, Hermes must not keep spinning and retrying; it must escalate and alert the project owner (Richy); only when Richy approves `ApprovedEquivalent` may an equivalent carrier be substituted, otherwise keep `Provisioning`/`NeedsAttention`/`PendingVerification` and wait for Richy to intervene.

### 7.3 Sandbox Tiers and Git Authorization (V3.0: unified danger-full-access + Code Immutability Constraint)

| Role | sandbox | Code Immutability Constraint |
|---|---|---|
| Implementer / Integrator / Doc·Design Reviewer | danger-full-access | Modify and commit within task scope |
| Coding / System / Final-Merge Reviewer | danger-full-access | **Code immutable**: only build, run tests, and perform read-only checks; must not modify business source, must not commit candidates, must not merge |
| Validator | danger-full-access | Same as Reviewer; verification artifacts (logs/evidence) writable |

The V2.5 read-only sandbox was empirically unable to compile/run tests, so Reviewers could not reach trustworthy conclusions; therefore, from V3.0 onward **all Codex dispatches uniformly use `danger-full-access`**. The permission expansion is offset by the "Code Immutability Constraint":

- Review must execute in an **isolated detached verification workspace based on the precise candidate commit**, protecting the developer's original worktree uncommitted content and Review independence;
- Verify the candidate HEAD/tree at review start; at review end verify again that HEAD is unchanged and the workspace has no tracked business-source diff—any change invalidates that Review's conclusion;
- Any issue found returns to the original development carrier as a Finding for rework; a Reviewer must not modify code itself to form a passing conclusion.

Hermes does not perform git on behalf of carriers; worktree creation is done by the Implementer (Hermes dispatches only TaskID/base branch/worktree directory), and cleanup is executed uniformly by the Integrator (see *Hermes Process & Boundary Resolution* C5/D2/D3b).

## 8. State Production and Consumption Protocol

Standard loop:

```text
Hermes reads the ledger
→ an event source arrives (push message / polling reaches completed) or scheduled reconciliation triggers
→ collect records with SignalState = PendingConsumption
→ only read the detailed final/help for that record's bound ExecutionRef
→ after verification, update the summary state, mark Consumed, and dispatch the follow-up
→ no new signal: only do ledger integrity and necessary Git health checks
```

Event consumption order:

1. Collect pending-consumption state records; when there is no `PendingConsumption`, do not scan the task system looking for results;
2. Compute idempotent identity by `RecordID + SignalRevision`;
3. Only read the original final/help content for the `ExecutionRef` bound to that record;
4. If already consumed, ignore idempotently;
5. Record `Received`, do not advance business state first;
6. Verify `ProtocolVersion`, `InvocationID`, `TaskID`, role, branch, and candidate identity;
7. Cross-verify facts using Git, immutable documents, or the corresponding authoritative source;
8. Mark `Validated`, `PendingVerification`, or `Rejected`;
9. Only `Validated` evidence may trigger normal state transitions;
10. Hermes writes the summary state, consumption confirmation, and follow-up stage record into the ledger, then increments `StateRevision`.

When cross-task reading fails, retain `PendingConsumption`, record the control-plane error, and notify; the next round still only processes that pending-consumption record, must not degenerate into scanning all execution carriers, and must not rewrite a read failure as business `Blocked`. When the same pending-consumption record fails to be read consecutively **3 times** and still cannot be processed, alert Richy per the §7.2 carrier-unavailable escalation threshold.

If the final lacks a structured protocol header but the content may be valid, first ask the same execution carrier to resend "protocol-header-only / missing-fields", do not directly negate the result or mechanically create a new task. When the execution carrier is unavailable, retain the original content and enter `PendingVerification`.

## 9. Evidence Attribution and Conflict Handling

Different sources only prove their own responsibility scope:

| Fact | Authoritative source |
|---|---|
| Task scope, dependencies, and authorization boundary | Approved task document |
| Commit, ancestry, and branch inclusion | Git |
| Whether development formally committed | Ledger record + Implementer final + Git verification |
| Review conclusion | Ledger record + independent Reviewer final |
| Which code the Review reviewed | Reviewer final + Git SHA |
| Whether the execution carrier is running or ended | Carrier-channel messages / polling event-source history |
| Project-owner authorization | Explicit user message or formal authorization record |

When sources conflict, record them side by side. For example, when code has appeared in the iteration branch but lacks Review/integration evidence:

```text
CodePresence: PresentInIteration
ProcessState: IntegrationUnverified
RequiredAction: audit source, candidate equivalence, and gate; do not auto-rollback or declare Integrated
```

## 10. Review Execution Status vs. Conclusion Separation

Reviewer output must separately contain:

```text
ExecutionStatus: Completed / Blocked
Verdict: Approved / ChangesRequested / NotIssued
BlockerType: None / RepositoryEnvironment / ToolRuntime / Authorization
```

Rules:

- Test `NotRun` does not automatically require `ExecutionStatus: Blocked`; the Reviewer may still form `ChangesRequested` based on a verifiable counter-example;
- Tool-call format, task reading, or model execution-protocol anomalies belong to `ToolRuntime`, not code Findings;
- Only when the review target cannot be parsed, repository objects are missing, etc., does it belong to `RepositoryEnvironment`;
- Unauthorized real external operations belong to `Authorization`;
- When `ExecutionStatus: Blocked`, `Verdict` must be `NotIssued`;
- `Approved`/`ChangesRequested` must bind the precise review target; a new `HeadSHA` automatically marks the old conclusion `Superseded`.

Test-responsibility layering (prevent token waste, per *Hermes Process & Boundary Resolution* §4):

- Level 0 is run once by the Implementer and dropped into the evidence directory;
- The Reviewer reviews on diff + L0 evidence, only smoke-tests key paths, **does not re-run the full unit suite**;
- Level 1 + regression is run once by the Validator only before merge, full-suite.

## 11. cron Reconciliation

Scheduled reconciliation each round (frequency registered in the instance capability record):

1. Read the ledger and `Paused`;
2. When `Paused=true`, only report the pause; do not read or dispatch business follow-ups;
3. Collect all `PendingConsumption` records;
4. Only read the bound `ExecutionRef` for pending-consumption records, verify and consume;
5. When there are no pending-consumption records, do not call the task-list or thread-read API; only check ledger integrity, document snapshots, necessary Git dependencies, and health;
6. Record the actually-read document version, state records, targeted execution carriers, and errors;
7. Continue waiting for the next round or event trigger;
8. **Stall detection and escalation** (see below).

### 11.1 Stall Detection

Cron reconciliation additionally checks each round for long-progress-less tasks, to avoid silent deadlock (note: tasks still active, with a recent heartbeat, or waiting for human authorization are not false-alarmed):

- A task continuously in `ContextGenerationPending` / `Ready` / `InProgress` / `PendingConsumption` for over **2 consecutive reconciliation cycles** (cycle length registered in the instance capability record) with no state advance or read activity → record stall and alert; still no progress over **4 reconciliation cycles** → escalate to the project owner (Richy).
- A task in `Integrated` but missing the `IntegrationVerified` writeback (i.e. no corresponding integration-verification writeback signal) for over **2 reconciliation cycles** → record stall and alert; over **4 reconciliation cycles** → escalate Richy, Hermes re-dispatches `IntegrationValidationTask` or a human intervenes.
- Dependency-cycle detection: when a task cannot advance because an upstream is long in `Ready`/`InProgress`, cron reports the dependency-blocking graph at stall escalation, to locate the cycle (the rejection (non-registration) rule for cycles is in `04` §5.2).

Do not report "no change" without the following minimal evidence:

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

Reconciliation must not only compare Git HEAD, nor re-dispatch instructions with an existing `DispatchKey`; when there is no `PendingConsumption`, do not read carrier-by-carrier just to "confirm no change".

## 12. Pause and Concurrency

### 12.1 Pause

When the project owner requests a pause:

- First set `Paused=true` and record the event;
- Stop new dispatch, rework, Review, integration, and business-state advancement;
- By default do not cancel already-running subtasks, unless the project owner explicitly requests;
- Subsequent arriving events may be saved as `Received` as-is, but no side effect is executed before recovery;
- Cron reconciliation only reports the pause state.

Recovery must record `Resumed`, and first perform unconsumed-event reconciliation.

### 12.2 Concurrency and Parallelism

- Hermes is single-machine, with exactly one current Coordinator owner per iteration, **no coordination-lease lock**; duplicate side effects are blocked by `DispatchKey` dedup and stale-Epoch fencing.
- Multiple tasks may run in parallel; tasks with conflicts (shared files / same worktree) are not set parallel; if a conflict actually arises, ask the project owner to coordinate (*Hermes Process & Boundary Resolution* C2).
- One task keeps only one valid execution instance at the same stage, unless split into non-conflicting subtasks.

### 12.3 Coordinator Ownership and Handover (V3.0)

- Every iteration has exactly one current Coordinator owner. The scheduling authority is fenced by `CoordinatorEpoch`; dispatch and signal consumption from a non-current Epoch **MUST** be rejected.
- `CoordinatorEpoch` is monotonically increasing. A takeover is valid only when the expected current Epoch and `StateRevision` still match at the atomic compare-and-swap point; otherwise it fails and the caller must re-read the ledger.
- A normal handover uses a unique `TransferID` and requires acknowledgement from both the old and new owner. The new owner may dispatch only after the handover is durably recorded.
- If the old owner is lost for the configured `LostOwnerTimeout` (default: two reconciliation cycles), recovery takeover still requires Richy's authorization. Timeout alone does not silently transfer ownership.
- Two active scheduling drivers for the same `IterationID` are prohibited. A second driver must remain read-only or stop; it may not dispatch, consume signals, or advance state.
- “Pause” changes only `Paused=true` and does not transfer ownership. “Resume scheduling” clears the pause and returns ownership to cron; interactive takeover requires an explicit owner-transfer instruction.

### 12.4 Carrier Policy and Change Control (V3.0)

- The **sole source of truth** for carrier-selection rules is the "Carrier Policy Artifact" (a versioned data file carrying `PolicyVersion` and `PolicyArtifactDigest`); the spec text no longer enumerates concrete carrier allocations.
- **Dual-layer change channel**:

| Change type | Example | Channel |
|---|---|---|
| Contract/schema change | Alters field structure, authorization model, fail-closed semantics, or acceptance logic | Spec change review (05 §6.3) |
| Instance-content change | Carrier temporarily unavailable, priority adjustment, quota change | Controlled runtime change: project owner announces → update artifact (version+1) → write a `PolicyChange` Signal in the ledger (recording authorizer / digest / scope of effect) → gates adopt the new version automatically |

- **Mandatory dispatch gate**: every dispatch must pass the pre-dispatch gate implementation check (fail-closed); items include at least: complete intent fields, globally unique RecordID (§6.2), carrier available and matching the policy artifact, single-thread constraint, scheduling pause gate, dependencies satisfied, and candidate SHA origin-reachable. The policy snapshot is recorded at task/stage level (§5.3): each dispatch's task record carries the then-effective PolicyVersion / PolicyArtifactDigest / DispatchedCoordinatorEpoch—iteration-level fields reflect only current state and cannot answer why a historical dispatch chose its carrier.
- When the gate verdict is BLOCKED, stop and escalate to the project owner; bypassing is prohibited.

## 13. Recovery Protocol (Three Tiers)

| Tier | Trigger | Hermes action | Failure handling |
|---|---|---|---|
| Hot recovery | One-off tool/call occasional anomaly | Log, auto-resume from the most recent ledger snapshot; retry | Consecutive retries **3 times** still fail → auto-escalate to cold recovery |
| Cold recovery | Process crash/restart | Fully auto-read ledger + Git rebuild `ConsumedRevision` → `RecoveryOnly` verification → resume | Failure auto-escalates to disaster tier |
| Disaster recovery | Ledger lost/corrupted (or cold recovery fails) | Rebuild from Git task-document snapshot, determine provable state, downgrade the rest to pending-verification | **Requires project-owner (Richy) intervention and approval** |

### 13.1 Recovery Escalation Threshold (explicit)

- **Hot → Cold**: the same hot-recovery action fails consecutively **3 times** and still cannot resume, Hermes auto-escalates to cold recovery, no infinite retry;
- **Cold → Disaster**: cold recovery (read ledger + Git rebuild + `RecoveryOnly` verification) fails, auto-escalate to disaster tier;
- **Disaster**: must wait for the project owner (Richy) to intervene and approve `RecoveryApproved` before exiting `RecoveryOnly`;
- Each escalation is recorded in the recovery report and ledger, to avoid silent infinite loops.

### 13.2 Cold Recovery Steps

1. Read the ledger; if missing or corrupted, recover the task graph, roles, dependencies, baselines, stable state, and unconsumed records from the last structurally-valid `TASK-STATE-EXCHANGE` snapshot in `CanonicalTaskDocumentPath`;
2. Scan Git branches, worktrees, commits, ancestry, and iteration-branch inclusion;
3. Only for stages recorded as `PendingConsumption`, `PendingVerification`, or missing evidence, discover the execution carrier by its `ExecutionRef`;
4. Only read the above target task history; forbid re-implementing cold recovery as a full cross-carrier scan;
5. Compute event fingerprints for final, help, commit, Review, rework, and integration;
6. Cross-verify by source responsibility and generate `RecoveredEvidence`;
7. Compute the maximum-safe state supported by evidence for each task;
8. Form a list of conflicts, missing items, wrong execution mechanisms, and unknown merges;
9. Write back to the ledger and save the old-state backup; at a stable gate point, copy the latest state as the development-task document persistent snapshot;
10. Output the recovery report; cold recovery **auto-completes on verification pass (no project-owner intervention needed)**, and exits `RecoveryOnly` resuming scheduling on verification pass.

### 13.3 Maximum-Safe State

- Only task-branch commit found: `CodePresence=PresentInTaskBranch`, `TaskState=InProgress` (existing state);
- Implementer final and Git both valid: recover at most to `Submitted`;
- Valid `ChangesRequested`: recover to `ChangesRequested`, and check whether a new Head already exists;
- Valid `Approved` and target equals current Head: may recover `TaskState=TaskAccepted`, but `RecoveryOnly` still forbids dispatching on this; recovery audit must be completed first;
- Current Head has changed: mark the old Review `Superseded`;
- Code already in the iteration branch but lacking compliant integration evidence: `IntegrationUnverified`, no auto-rollback, no declaring `Integrated`;
- Review history lost: re-Review, do not re-develop verifiable commits;
- Authorization record lost: re-obtain authorization, do not infer authorization existence from code results.

Recovery events must carry `Recovered=true`, source, original time, recovery time, and confidence: `Verified / Strong / Partial / Unknown`. Only `Verified` or project-spec-explicitly-allowed `Strong` may support normal state transitions.

## 14. Protocol Canary and Acceptance

Before launching a real high-risk iteration, the control plane must be tested in a no-business-side-effect scenario, covering at least:

1. Dispatch returns only a temporary request identifier, can correctly enter `Provisioning` and bind a formal `ExecutionRef`;
2. The subtask completes earlier than the cron reconciliation cycle, and can be consumed on event or next reconciliation;
3. Push event sources unavailable, but when the ledger has `PendingConsumption`, the result can still be precisely located;
4. When final is missing fields, ask the original task to resend, do not repeatedly create an execution instance;
5. Re-reading the same `RecordID + SignalRevision` does not re-dispatch;
6. When the user pastes complete results, enter verification rather than directly negate or directly approve;
7. Tool runtime errors enter `ControlPlaneError`, do not pollute business `Blocked`;
8. A new `HeadSHA` invalidates the old Review;
9. After `Paused=true`, no new side effect is produced;
10. Hermes can hot-recover from only the ledger and necessary Git facts; if the ledger is lost, can cold-recover from the development-task document snapshot;
11. After deleting the derived evidence directory, the ledger is not lost and scheduling can continue;
12. When a designated execution carrier fails, it does not trigger another execution mechanism as fallback (unless `ApprovedEquivalent` authorized);
13. Requirement, design, and task baselines parsed from different worktrees are consistent;
14. Sandbox tiers are correct: workspace-write blocks git, danger-full-access allows git;
15. Cron fallback can still reconcile and consume after event loss, without producing duplicate side effects.

When Canary fails, real business tasks stay `Planned/Ready`; do not use real development to verify the scheduling protocol.

### 14.1 Canary Failure Handling (CanaryFailed)

When any Canary scenario fails, enter the `CanaryFailed` state, and close the loop per the path below, to avoid tasks silently stuck at `Planned/Ready` unnoticed:

1. Record the failed scenario number, phenomenon, and root cause (control-plane defect / carrier anomaly / protocol mismatch, etc.);
2. The responsible party (Hermes engineering side) fixes the control plane or carrier config;
3. Re-run Canary → on pass, lift the real-business-task `Planned/Ready` dispatch freeze;
4. Consecutive **3 times** re-run still not passing → alert the project owner (Richy) for manual intervention, do not keep auto-retrying to hide the problem;
5. Throughout, notify Richy of the current `CanaryFailed` state and handling progress via configured notification channels.

`CanaryFailed` is not business `Blocked`, and does not pollute code-task state; it only freezes the launch of real high-risk iterations until Canary passes or Richy explicitly records a temporary waiver.

## 15. Review Checklist

- [ ] Execution-carrier state, task state, and evidence state are separated;
- [ ] Unique `LedgerLocation`, `CanonicalTaskDocumentPath`, and Schema are explicit; the ledger is not in Git, and `TASK-STATE-EXCHANGE` is the persistent snapshot;
- [ ] The task has Invocation, `ExecutionRef`, and execution-carrier state;
- [ ] All dispatches have idempotent `DispatchKey` and causal events;
- [ ] Code and document baselines are unambiguous;
- [ ] State consumption uses `RecordID + SignalRevision`, and scans no execution carrier when there is no pending-consumption record;
- [ ] A dispatch request identifier being `Provisioning` success is not treated as failure;
- [ ] The project-required execution mechanism may not be substituted without authorization;
- [ ] Review execution status and Verdict are separated;
- [ ] Sandbox tiers are correct, roles needing git use danger-full-access;
- [ ] Cron reconciliation can prove it actually read the ledger, and only targeted reads the execution carrier for `PendingConsumption` records;
- [ ] Pause and `DispatchKey` idempotent dedup can prevent duplicate side effects;
- [ ] Hot, cold, and disaster recovery all have explicit gates;
- [ ] Canary covers duplicate state, early completion, event loss, cross-worktree unification, sandbox tiers, and concurrent reconciliation;
- [ ] Recovery results must be reviewed before re-dispatch.

## 16. Definition of Done

Hermes may schedule a real iteration only when the project has configured the Hermes ledger, immutable execution baselines, legal state transitions, idempotent dispatch, pause, three-tier recovery, and Canary, and has passed project-owner review.

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.5 (pending review) | 2026-08-20 | WorkBuddy | Renamed + rewritten from V2.4 "09-Scheduling Control Plane & Runtime Ledger Spec" into the resident-Coordinator (Hermes) mode (ledger schema / three state types / DispatchKey idempotency / event+cron dual channel / single-instance no-lease / ExecutionRef / three-tier recovery / sandbox tiers) |
| V2.5 (pending review) | 2026-08-20 | Hermes | Added author/reviewer/revision metadata |
| V2.5 (pending review) | 2026-08-20 | Hermes | Review revision: ledger must never be committed to Git (prevent circular reference / self-contained hash); cold recovery auto-completes on verification pass without Richy; EvidenceIncomplete changed to existing state InProgress |
| V2.5 (pending review) | 2026-08-20 | Hermes | Added 7 process gaps: §3.2 added ContextGenerationPending; §6.1 added CI/integration-verification writeback + candidate-freeze human-gated event; §7.2/§8 carrier-unavailable 3-consecutive escalation to Richy; §11.1 stall detection (incl. IntegrationVerified timeout escalation); §13.1 recovery escalation threshold (hot→cold→disaster, hot 3 times); §14.1 CanaryFailed state + handling path + 3-consecutive alert Richy |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
| V2.5 errata | 2026-08-21 | WorkBuddy | Synced source errata bd6a71f: heading-level, wording, and reconciliation-terminology fixes |
| V3.0-draft | 2026-08-24 | Hermes | Added the Carrier Policy Artifact contract, PolicyArtifactDigest, CoordinatorEpoch fencing, atomic conditional takeover, TransferID, LostOwnerTimeout, and the single-owner / dual-driver prohibition; unified Codex dispatch on danger-full-access and added the Code Immutability Constraint |
| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all review rounds closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
| V3.0 sync | 2026-08-25 | Hermes | WB batch review closure: added §12.4 (carrier policy and change control); §5.2/§5.3 synced with Chinese V3.0 mandatory fields (CoordinatorEpoch / PolicyVersion / PolicyArtifactDigest / MaxAutomaticAttempts etc.); §6.1 event sources abstracted to functional types; §6.2 RecordID global uniqueness (ULID/UUID); §7.1 dispatch adapter contract; §7.3 role-based immutability table; deployment-fact residuals removed |
