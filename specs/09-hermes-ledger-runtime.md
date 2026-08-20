# Hermes Ledger & Runtime Spec

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Applicability: development iterations led by the resident Coordinator (Hermes), with WorkBuddy / Codex / Human collaborating in parallel
> Predecessor: `09-Scheduling Control Plane & Runtime Ledger Spec` (V2.4, Codex thread as Coordinator)
> Revised: 2026-08-20
> Foundation: *Hermes Capability Boundary List* (measured capability), *Hermes Process & Boundary Resolution* (boundary decisions)
> Author: WorkBuddy (rewrite)
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
7. **Single-instance idempotency**: Hermes is single-machine, single-instance, with no competing Coordinators; idempotency is guaranteed by `DispatchKey` dedup, with no coordination-lease lock.
8. **Event-driven + cron fallback**: events (Feishu long-connection / Codex gateway polling) take priority, cron periodic reconciliation is the fallback; lost events do not affect correctness.
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
| Completed | Execution ended; a pending-consumption signal must be read before judging the task conclusion |
| Unavailable | Currently unreadable or the execution carrier is lost |
| Cancelled | The execution carrier has been explicitly cancelled |

When the Codex gateway returns a formal `thread id`, register the execution identifier directly; if only a request identifier is returned, that means `Provisioning`, and must not be judged as creation failure. Use `ExecutionRef` uniformly to reference the execution carrier, not distinguishing `clientThreadId/threadId`.

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

- **The Hermes ledger** (`LedgerLocation`, local JSON state file + optional SQLite, **inside the project directory, not committed to Git**) is the **sole real-time source of truth for task runtime state** during the active iteration, maintained by the single Hermes instance. **The ledger must never be committed to Git—even if the execution carrier has `danger-full-access` commit permission, no add/commit may include ledger files** (the ledger changes with development and would create a circular reference / self-contained hash with commits, and must be isolated from the version repository).
- **The development-task document** (`CanonicalTaskDocumentPath`) `TASK-STATE-EXCHANGE` block is the **Git persistent snapshot**, used for cross-restart / disaster recovery.
- The execution Agent **does not write the ledger directly**; it reports structured results through the carrier channel, which Hermes consumes and writes into the ledger (see §8).
- The ledger stores only summaries, SHAs, Verdict, `ExecutionRef`, and next-action needed for locating and verifying; long logs, full diff, and full final are not written into the ledger, but fall into the derived evidence directory.

### 5.2 Iteration-Level Minimal Fields

```text
SchemaVersion, ProtocolVersion, IterationID, HermesInstanceRef,
CoordinatorMode, Paused, CodeBaseSHA, RequirementsBaselineRef,
DesignBaselineRef, TaskDocumentBaselineRef, CanonicalTaskDocumentPath,
LedgerLocation, StateRevision, ConsumedRevision, UpdatedAt
```

### 5.3 Task-Level Minimal Fields

```text
RecordID, TaskID, TaskType, Stage, InvocationID, ParentIterationID, Role,
ExpectedExecutionKind, ExpectedModel, ActualModel, ModelProvider, Sandbox,
TaskBranch, WorktreePath, CodeBaseSHA, HeadSHA,
ExecutionRef, CarrierStatus, TaskState, EvidenceState,
SignalRevision, SignalState, ProducedAt, ConsumedAt, ConsumedBy,
LastEventFingerprint, BlockerType, RecoveryConfidence, NextAction
```

`ExpectedModel` is what the project requires; when `ActualModel` cannot be verified, record `Unknown`, and do not claim a model match on your own.

## 6. Events, Idempotency, and Dispatch Dedup

### 6.1 Event Sources and Dual-Channel Scheduling

| Channel | Coverage | Nature |
|---|---|---|
| Feishu long-connection (lark-ws) | Receive WorkBuddy reply messages | Event, may be lost (empirically confirmed) |
| Codex gateway polling `GET /v1/threads/:id` | Query Codex thread progress | Event, active polling |
| CI / integration-verification writeback | After the Integrator merges the precise commit, it runs/triggers affected integration checks and reports a structured protocol header (with `IntegrationCommit` + `IntegrationStatus`) through its carrier channel (Feishu/Codex), which Hermes consumes into the ledger | Event, reported by the execution carrier, not depending on an external webhook |
| Candidate freeze (human-gated) | After the project owner freezes the iteration candidate and informs Hermes (or Hermes polls the freeze marker), it triggers dispatching Level 1 (Validator / System Reviewer) | Event, human-gated |
| Hermes cron periodic polling | **Fallback reconciliation** (~1 minute) | Fallback, covers event loss |
| TG push | Notify key nodes | Output, 0 token |

**Conclusion**: events (Feishu + Codex polling + CI writeback + candidate freeze) are consumed first, cron periodic reconciliation is the fallback; lost events do not affect correctness.

> **IntegrationVerified writeback note**: after `Integrated`, `IntegrationVerified` no longer depends on an external CI webhook. The execution subject is the **Integrator** (or an independent `IntegrationValidationTask`): after it merges the precise commit, it runs/triggers affected integration checks and reports a structured protocol header (with `IntegrationCommit` + `IntegrationStatus=Passed/Failed`) through its own carrier channel (Feishu/Codex); Hermes consumes this signal into the ledger and triggers downstream Level 1. A missing report is treated as a pending-consumption signal, escalated by cron stall detection (see §11).

### 6.2 Idempotent Identity

- Dispatch idempotency key: `DispatchKey = IterationID + TaskID + Stage + TargetIdentity`. When the same `DispatchKey` already has a valid instance, re-dispatch is forbidden.
- Consumption idempotency key: `RecordID + SignalRevision`. Re-reading the same record only updates read metadata, and does not repeatedly create Review, rework, or integration tasks.
- Event fingerprint `Fingerprint` covers at least event type, task, Invocation, target SHA/candidate, and normalized result.

### 6.3 Causality

Any new dispatch that produces a side effect must record:

```text
CausedByEventID
DispatchKey = IterationID + TaskID + Stage + TargetIdentity
```

## 7. Dispatch and Identity Binding

### 7.1 Dispatch API

Hermes → execution carrier:

| Carrier | Dispatch method | Sync |
|---|---|---|
| WorkBuddy | Feishu post message @WB | Async (one-way delivery) |
| Codex | `POST /v1/threads`, body `{prompt, cwd, model, modelProvider, sandbox, approval}` | Sync / async |
| Human | Hermes notifies via TG/Feishu, human performs the work | Async |

`modelProvider`: `openai` (native) / `opencodex` (CodexSplit third-party).

### 7.2 ExecutionRef Unified Identity

- All execution carriers are uniformly modeled as `ExecutionRef`, replacing the old `clientThreadId → threadId` two-stage mapping.
- When Codex returns a formal thread id, register `ExecutionRef = thread id`; when only a request identifier is returned, register `Provisioning`, and resolve the formal identifier later.
- When a dispatch-create call fails, register `ControlPlaneError`, do not register business `Blocked`, and do not fall back to another execution mechanism (unless the project owner approves `ApprovedEquivalent`).
- **Carrier-unavailable escalation threshold**: if the same execution carrier (identified by `ExecutionRef`) fails dispatch/read consecutively **3 times** and remains unavailable, Hermes must not keep spinning and retrying; it must escalate and alert the project owner (Richy); only when Richy approves `ApprovedEquivalent` may an equivalent carrier be substituted, otherwise keep `Provisioning`/`NeedsAttention`/`PendingVerification` and wait for Richy to intervene.

### 7.3 Sandbox Tiers and Git Authorization

| Task type | sandbox | Note |
|---|---|---|
| Read-only investigation / code review | read-only | Reviewer, read-only investigation |
| Write document / write evidence | workspace-write | Does not touch `.git` |
| **Needs git commit** (development / integration (incl. merge) / doc design) | danger-full-access | `.git` is a protected path under workspace-write |

Roles needing `git add/commit/merge/push` (Implementer, Integrator, Doc/Design Reviewer—the latter needs to commit design/work-package/context documents) use `danger-full-access`. Reviewer and Validator are read-only sandboxes. Hermes does not perform git on their behalf; worktree creation is done by the Implementer (Hermes only dispatches TaskID/base branch/worktree directory), review is entered read-only by the Reviewer, and cleanup is uniformly executed by the Integrator (see *Hermes Process & Boundary Resolution* C5/D2/D3b).

## 8. State Production and Consumption Protocol

Standard loop:

```text
Hermes reads the ledger
→ event arrives (Feishu receives WB reply / Codex gateway polls completed) or cron reconciliation triggers
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
| Whether the execution carrier is running or ended | Feishu message / Codex gateway thread history |
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

## 11. cron Reconciliation (Periodic Reconciliation)

Hermes cron (~1 minute) reconciliation each round:

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

- A task continuously in `ContextGenerationPending` / `Ready` / `InProgress` / `PendingConsumption` for over **2 cron cycles** (~2 minutes) with no state advance or read activity → record stall and alert; still no progress over **4 cron cycles** → escalate to the project owner (Richy).
- A task in `Integrated` but missing the `IntegrationVerified` writeback (i.e. no corresponding CI/integration-verification writeback signal) for over **2 cron cycles** → record stall and alert; over **4 cron cycles** → escalate Richy, Hermes re-dispatches `IntegrationValidationTask` or human intervenes.
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

- Hermes is single-machine, single-instance, with no competing Coordinators, **no coordination-lease lock**; duplicate side effects are blocked by `DispatchKey` idempotent dedup.
- Multiple tasks may run in parallel; tasks with conflicts (shared files / same worktree) are not set parallel; if a conflict actually arises, ask the project owner to coordinate (*Hermes Process & Boundary Resolution* C2).
- One task keeps only one valid execution instance at the same stage, unless split into non-conflicting subtasks.

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

1. Read the ledger; if missing or corrupted, recover the plan graph, roles, dependencies, baselines, stable state, and unconsumed records from the last structurally-valid `TASK-STATE-EXCHANGE` snapshot in `CanonicalTaskDocumentPath`;
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
3. Feishu events unavailable, but when the ledger has `PendingConsumption`, the result can still be precisely located;
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
5. Throughout, notify Richy of the current `CanaryFailed` state and handling progress via TG / Feishu.

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
