# Development Task Spec

> Spec version: V3.0
> Document status: Approved (V2.5 is the previous baseline; finalized 2026-08-24, Richy final review passed)
> Document positioning: transforms the approved design into dispatchable, reviewable, combinable-verifiable development tasks
> Prerequisite gate: applicable requirements and detailed design have been approved; exploration tasks excepted
> Author: Tiffany-Dev (V3.0 revision; initial version = WorkBuddy)
> Originally finalized: 2026-08-11
> Revised: 2026-08-24
> Reviewer: Richy (approved)

## 1. Single Responsibility

This document only specifies:

- Which development track is adopted;
- How to split tasks, express dependencies, and control parallel conflicts;
- Each task's goal, authorization boundary, risk, deliverables, and status;
- Who owns cross-task system integration;
- The project's actual roles, branches, and verification configuration.

The following are owned as the single source of truth by other specs; this document only references them:

- Context L1/L2/L3: `06-context-package.md`;
- Agent roles and prompts: `07-subagent-delegation-prompts.md`;
- Tests, Review, Level 0–3, and merge eligibility: `08-verification-review-merge-gate.md`;
- Scheduling control plane, runtime ledger, reconciliation, and recovery: `09-hermes-ledger-runtime.md`;
- Review, change, and evidence validity: `05-doc-review-change-control.md`.

Do not re-copy the full rules or templates of those specs into this document.

## 2. Development Track Selection

The project selects from the following three tracks. The track is decided jointly by failure consequence, impact scope, reversibility, concurrency complexity, data or security risk, external uncertainty, observability, and test gap—not merely by lines of code or technical category.

### 2.1 Lightweight Track

For local, reversible, small-impact, adequately-automated changes, e.g. simple defects, pure logic, or low-risk config adjustments.

Minimum artifacts:

- A task brief, recorded directly in an Issue, task document, or approved equivalent carrier;
- Goal, scope, risk, verification, and rollback description;
- Developer self-check and an independent second-view Review;
- No mandatory independent Context L2, and no mandatory full four-document chain.

### 2.2 Standard Track

For single-module or few-module development with controllable risk but needing explicit design and integration verification.

The background, requirements, design, and tasks may be merged into one structurally-complete development description, but the core questions of each stage, SOLID review, test levels, and approval status must be retained.

Context L2 is generated or not per `06`'s trigger condition.

### 2.3 High-Risk Track

For cross-module, multi-agent, long-cycle, complex-concurrency, production-entry, critical-persistence, migration, security, irreversible-operation, external-protocol, or obvious-test-gap tasks.

Use the full document chain, formal task graph, independent Context L2, explicit system-integration tasks, and a Level 0–3 verification plan.

### 2.4 Track Upgrade and Downgrade

- The project may pre-approve a default track and risk threshold to avoid re-approving every small task;
- When impact expands, assumptions fail, or a high-risk junction appears, the track must be upgraded;
- Downgrade must record its basis; it must not be decided by the developer merely to reduce gates;
- Simplifying the track does not exempt requirements consistency, SOLID, evidence authenticity, or external-operation security.

## 3. Exploration Tasks

When key facts are insufficient to complete requirements or design, a technical Spike or exploration task may be established first:

- Read-only by default, or executed in an isolated environment;
- Produces no directly-mergeable formal feature code; if experimental code is produced, it should be explicitly discarded or re-enter the formal development flow;
- Outputs facts, limits, risks, candidate approaches, and what remains unknown;
- Must not use exploration to bypass requirements, design, or external-operation authorization;
- After exploration conclusions are written back to background or design, the formal development track is decided.

## 4. Project-Level Execution Configuration

The applicable configuration should be recorded before splitting. Low-risk projects may reference a long-lived project configuration instead of copying it each time.

```text
DevelopmentRoute: Lightweight / Standard / HighRisk
IterationID: <stable iteration ID>
IterationBranch: <project actual value or N/A>
IntegrationBranch: <project actual value or N/A>
StableBranch: <project actual value or N/A>
BranchStrategy: <project strategy; default mapping see 08>
TaskBranchPattern: <e.g. <iteration>-<task-id>>
ImplementerProfile: <capability requirement and execution instance>
CodingReviewerProfile: <capability requirement and execution instance>
IterationIntegratorProfile: <when applicable>
SystemReviewerProfile: <when applicable>
MainMergeExecutorProfile: <when applicable>
Level2ExecutionMode: Manual / SemiAutomatic / Automatic / N/A
Level3ExecutionMode: Manual / SemiAutomatic / Automatic / N/A
DestructiveResourcePolicy: <authorization boundary>
CanonicalTaskDocumentPath: <Git-managed task-definition and persistent-snapshot path>
LedgerLocation: <Hermes ledger location; local JSON state file + optional SQLite, not in Git>
DispatchMode: EventDriven + CronFallback
RuntimeLedgerLocation: <derived evidence/audit cache location not entering candidate hash / N/A + reason>
CoordinatorProtocolVersion: <version>
RecoveryPolicy: Hot + Cold + Disaster / N/A
CanaryRequirement: Required / N/A + reason
ContextL1Location: <project actual location or N/A>
ContextL2Policy: RiskTriggered / Required
```

Model, vendor, branch name, directory, and reconciliation period belong to project configuration, not generic spec constants.

## 5. Task-Splitting Principles

### 5.1 Atomicity

A task should have one concern and a set of highly-related changes, completable in one development–Review loop. If it simultaneously contains multiple independent state machines, external systems, or unrelated refactors, split further.

Do not over-split into tasks that only produce empty files or no independent behavior.

Tasks must first be classified by responsibility; the same task must not simultaneously represent coding, iteration merge, integration validation, and main-branch merge:

| TaskType | Responsibility | Typical deliverable |
|---|---|---|
| CodingTask | Independent coding, directed verification, and commit to task branch | `CodeBaseSHA..HeadSHA` |
| ReworkTask | Close Review Findings on the original task branch | new `HeadSHA` |
| IterationMergeTask | Merge the precise approved task commit into the iteration branch, or promote a non-main candidate per passed gate | `Integrated` / candidate-promotion commit |
| IntegrationValidationTask | Execute Level 1 / system verification on the iteration candidate | `IntegrationVerified` or Findings |
| MainMergeTask | After final gate and authorization, merge the precise candidate into the main branch | `Merged` commit |
| DocumentationTask | Update spec, design, evidence, or context | documentation review target |

`IterationMergeTask`, `IntegrationValidationTask`, and `MainMergeTask` must not smuggle in new business implementation. When code change is needed, return to the original CodingTask/ReworkTask or create a new fix task.

### 5.2 Dependencies and Continuous Integration

- Stabilize the truly-shared contract first, then implement the tasks depending on it;
- Each dependency must be labeled with a type: `Code`, `Review`, `Integration`, `Validation`, or `Evidence`;
- Downstream must not treat an unreviewed implementation as a formal baseline; a downstream task that actually needs upstream code defaults to depending on `Integrated`; it may depend on `TaskAccepted` only when it consumes a stable contract and needs no implementation;
- The pre-verification and post-integration re-verification of the same task must use different IDs or explicit sub-stages, e.g. `DEV-008A` and `DEV-008B`; do not reuse the same status to create a circular dependency;
- **Dependency-graph cycle detection**: Hermes performs a DAG check when registering task dependencies in the ledger; if a cycle (A↔B mutually satisfied) is detected, it **rejects the task registration and escalates to the project owner (Richy)**, rather than silently registering it so the task can never reach `Ready`. Stagnation detection and escalation for long-lived `Ready`/`InProgress` with no progress: see `09` §11.1;
- Each accepted task, after being merged into the iteration candidate by an independent `IterationMergeTask`, should run affected continuous-integration checks;
- Formal Level 1 still forms a version conclusion on the frozen candidate, but integration must not wait until the end of the iteration.

### 5.3 Change Authorization and File Ownership

- Authorize by default by responsibility, module, or behavior boundary, allowing modification of directly-related tests and local wiring;
- Parallel conflict, shared entry, sensitive resources, and high-risk tasks require a precise file list;
- Read-only investigation scope should cover callers, production entry, registration, config, tests, and adjacent modules;
- Modifications beyond the responsibility boundary must be escalated first;
- The user's existing changes and unrelated untracked files must be preserved.

#### 5.3.1 Branch, worktree, and Git Permissions

- Every real `CodingTask` must have a unique task branch and an independent worktree; the worktree is only a working directory, the branch is the task's ownership, the commit SHA is the review identity;
- The developer and rework executor may only create local commits on their own task branch; they must not directly commit or merge into the iteration development branch, long-term integration branch, stable branch, or main branch;
- Coding Reviewer, System Reviewer, and final-merge Reviewer are subject to the **candidate-content immutability constraint**: in a danger-full-access sandbox they may still build, test, and write evidence, but must not make a candidate pass by modifying it;
- Only an independent `IterationMergeTask` may merge the precise `TaskAccepted` commit into the iteration development branch;
- Only an independent `MainMergeTask` may merge the precise candidate into the main branch after `MergeApproved` and explicit project-owner authorization;
- Rework continues on the original task branch and produces a new commit; the new `HeadSHA` automatically invalidates the old Review conclusion.

### 5.4 Cross-Task System Integration Responsibility

When multiple tasks or a critical end-to-end path are involved, a system-integration responsibility task or owner must be designated, to prevent all local tasks passing while no one verifies the combined behavior.

Cover by risk:

- Production entry and composition root;
- Cross-task interfaces and config;
- Lifetime, Scope, concurrency, and shared resources;
- Error propagation, runtime state, and partial failure;
- Persistence or external boundary;
- Continuous-integration checks and formal Level 1 evidence.

When a single local task has no real junction, mark `N/A`.

## 6. Risk Rating

`Low / Medium / High` is recommended; judging basis includes:

- Failure consequence and impact scope;
- Reversibility of the change and rollback difficulty;
- Data-corruption, security, or permission risk;
- Concurrency, timing, and state complexity;
- Uncertainty of external dependencies;
- Observability and fault-discovery speed;
- Proof gap between automated tests and the real environment.

Technical categories such as DI, Host, persistence, and external services can only trigger risk assessment, not alone decide `High`.

Risk rating decides the development track, Context L2, Review capability, and verification depth.

## 7. Single-Task Contract

Every formal development task contains at least the following; the Lightweight track may use a compact table.

### 7.1 Identity and Baseline

- Stable ID, name, `TaskType`, track, and risk level;
- Source requirements, design decisions, and acceptance items;
- `InvocationID`, expected execution mechanism, and execution role;
- `TaskBranch`, `WorktreePath`, `CodeBaseSHA`, expected `ReviewedCommitRange`/`ReviewedCommitSet`, and `MergeTarget`; pure-document or approved-exception may use an unambiguous document target;
- `RequirementsBaselineRef`, `DesignBaselineRef`, `TaskDocumentBaselineRef`, and applicable `ContextBaselineRef`; the document baseline must not only reference another mutable workspace path;
- Dependent tasks, `DependencyType`, required state, and parallel condition;
- Context L2 path, or `N/A` with trigger judgment.

### 7.2 Goal and Scope

- One-sentence goal and user/system result;
- Authorized responsibility, module, or precise file;
- Read-only scope that must be investigated autonomously;
- Explicitly prohibited behavior and external operations;
- Non-task content.

### 7.3 Implementation and Verification Responsibility

- Behaviors, interfaces, config, or scripts that must be delivered;
- Normal, boundary, failure, cancellation, and compatibility requirements;
- Level 0–3 evidence owned by this task;
- Applicable production entry, lifetime, and resource risk;
- `NotRun`, environment limits, and subsequent owner.
- External side-effect risk `ExternalSideEffectRisk` and safe-validation method `SafeValidationMethod`;
- Who bears directed verification on the task branch, integration verification on the iteration branch, and manual verification by the owner respectively.

Any verification that may change database, index, storage, or real external state must not be executed by `CodingTask`, `ReworkTask`, Reviewer, or merge tasks; it must be split into an independent `IntegrationValidationTask`/external-verification task, with project-owner authorization and execution subject recorded.

### 7.4 SOLID Hard Gate

Every task must undergo SOLID judgment:

- S: is the responsibility and reason-to-change clear;
- O: does extension avoid unnecessary modification of the stable core;
- L: are the behavior, error, and cancellation semantics of the same abstraction substitutable;
- I: does the caller depend only on the capability it needs;
- D: does the core logic depend on abstractions, with technical details at the boundary.

Low-risk tasks may submit a concise "no material risk" conclusion, but the Reviewer must still complete the judgment. High-risk, trade-off, or detected-violation cases must expand evidence item by item. SOLID must not be exempted due to track simplification, existing-code issues, or passing tests.

### 7.5 Completion Evidence

- Actual review target: at least `CodeBaseSHA`, `HeadSHA`, and `CodeBaseSHA..HeadSHA`;
- Change summary and files;
- Build and test commands, environment, and results;
- Important findings from autonomous exploration;
- Not-run items and proof gaps;
- Independent Review conclusion;
- Whether it affects upstream documents, Context L1, or other tasks.

A coding task may not enter `InReview` before creating a task branch and delivering a commit. Pure-investigation, pure-document, or specially-approved tasks may not create a code commit, but must declare the candidate form and reason in the task contract beforehand.

### 7.6 Duration and Critical Path

Estimation must not count only coding time. At least list separately:

- Implementation and self-check;
- Coding Review;
- One expected rework round;
- Iteration merge;
- Integration verification and system Review;
- Evidence consolidation, manual gate, and external waiting.

Also give ideal, normal, and conservative (including one rework round) effort, marking parallelizable parts and the serial critical path.

## 8. Task State and Unified State-Exchange Surface

### 8.1 Sole State Source of Truth

The project must specify both the Git-managed detailed development-task document and the Hermes ledger (`LedgerLocation`). The detailed development-task document holds the task definition and the most recent persistent snapshot; the Hermes ledger (not in Git) is the sole real-time source of truth for task runtime state in the active iteration, **maintained by the current CoordinatorEpoch owner** (concurrency model: 09 §12.3).

`TaskDocumentBaselineRef` continues to identify the immutable task definition. Routine changes in the state-exchange area do not change the requirements, design, or code-review baseline of a dispatched task; any content change outside the state-exchange area still follows the document-change flow to judge whether it invalidates Context, Review, or dispatch.

Hermes must no longer use cross-task API, UI state, or chat context as the task-state discovery entry. When the ledger is missing, recover from the persistent snapshot in the detailed development-task document and Git; do not overwrite a higher `StateRevision` in the ledger with an old snapshot.

### 8.2 Ledger Contract Reference (V3.0: schema no longer duplicated)

The detailed development-task document **no longer duplicates the ledger field lists**. The runtime ledger—including all iteration-level and task/stage-level required fields (including CoordinatorEpoch, PolicyVersion, PolicyArtifactDigest, MaxAutomaticAttempts, etc.)—must conform to the **09 §5 "Ledger State Surface" runtime-ledger contract**; this task's `TASK-STATE-EXCHANGE` block is only the most recent persistent snapshot, not a runtime write target.

State-semantics constraints (aligned with `09`):

- `SignalState` uses only: `None`, `PendingConsumption`, `Consumed`, `Superseded`; the execution agent reports structured results through its carrier channel; after Hermes consumes them it increments `SignalRevision` and `StateRevision` and writes `PendingConsumption`; only after Hermes completes verification and follow-up actions is it changed to `Consumed`. The same `RecordID + SignalRevision` is the idempotent consumption key;
- Carrier-selection basis is recorded as PolicyVersion / PolicyArtifactDigest / DispatchedCoordinatorEpoch (task level; see 09 §5.3).

### 8.3 Write and Idempotency Rules

- The execution agent **does not write the ledger directly**; it outputs a structured protocol header through its carrier channel, and Hermes consumes it and writes to the ledger idempotently (event + cron fallback);
- Subtask start, help, completion, Review conclusion, rework commit, and integration result must first output a structured protocol header, then the final; the final is detailed evidence, not a state-discovery entry;
- A subtask may only report its own status record; it must not update other tasks' summary status, approve its own implementation, or dispatch subsequent tasks;
- Hermes is the sole writer of summary status and consumption confirmation; upon finding `PendingConsumption`, it reads the corresponding execution carrier by the precise `ExecutionRef` in the record;
- Ledger write idempotency is guaranteed by `DispatchKey = IterationID + TaskID + Stage + TargetIdentity` and `RecordID + SignalRevision` deduplication; scheduling-authority uniqueness is guaranteed by the CoordinatorEpoch FencingToken (09 §12.3, V3.0);
- Every dispatch records PolicyVersion / PolicyArtifactDigest / DispatchedCoordinatorEpoch (task-level fields);
- On state-write failure, do not claim the state was published; Hermes's next reconciliation relies only on the ledger, not a full scan of all execution carriers;
- The ledger must not store long logs, full diffs, or large finals; it stores only the summary, SHA, Verdict, execution reference, and next action needed for locating and verifying.

### 8.4 Stable Task States

The task summary state uses:

| State | Meaning |
|---|---|
| Planned | Planned, not yet startable or started |
| Ready | Dependencies, branch, baseline, and dispatch checklist satisfied, can start |
| InProgress | Developing or reworking |
| Submitted | Task branch has produced a delivery commit, awaiting independent Review |
| InReview | Submitted to independent Review |
| ChangesRequested | Reviewer has raised Findings to close, awaiting authorized rework |
| TaskAccepted | Level 0 passed |
| MergePending | Passed Level 0, awaiting independent iteration-integration task |
| Integrated | Precise approved commit merged into the current iteration branch and push succeeded |
| IntegrationVerified | Affected integration checks after merge passed |
| Blocked | Has a clear block |
| Cancelled | Approved cancellation |

Transient heartbeats and tool-call details need not be written to the Git document every round; but `InvocationID`, `ExecutionRef`, current `TaskState`, pending-consumption signal, target SHA/Verdict, and produce/consume times must be written to the ledger idempotently by Hermes.

`TaskAccepted`, `Integrated`, and `IntegrationVerified` cannot substitute for each other. `ComponentVerified`, `SystemVerified`, `ExternalVerified`, and `MergeApproved` belong to candidate versions, not single-task states.

## 9. Scheduling and Rework

- One task may have only one valid development instance at the same stage, unless split into non-conflicting subtasks;
- Each dispatch must have a unique `InvocationID`, idempotent `DispatchKey`, and immutable code/document baseline;
- After development commit, enter independent Review;
- `ChangesRequested` returns to the original task and original developer or approved substitute;
- Authorized rework continues with the same task ID and original task branch, producing a new `HeadSHA` and invalidating the original Review conclusion;
- Only after Review passes and evidence is valid does the Coordinator update `TaskAccepted`;
- After `TaskAccepted`, an independent iteration-integration task merges the precise `HeadSHA`; the developer must not merge it themselves;
- After dispatch, Hermes consumes the ledger via event sources + scheduled reconciliation fallback; only upon finding `PendingConsumption` does it read that record's `ExecutionRef` and process immediately;
- Cron reconciliation only reads the Hermes ledger and necessary Git facts; it does not iterate the task list, read carrier-by-carrier, or depend on cross-task API events;
- Task creation, document-state production/consumption, pause, resume, and Canary follow `09-hermes-ledger-runtime.md`; when the ledger is unavailable, follow the recovery protocol and do not continue dispatching;
- The project-explicitly-specified execution mechanism must not be substituted by another mechanism without authorization (no pseudo-independent self-review);
- Reconciliation must not treat a single compile/test failure or unfinished code as final block. Interruption is allowed only when the executor explicitly reports a block, an authorization/scope conflict is found, or continuous checks confirm no activity and no recoverability;
- When the same Finding is unclosed for two consecutive rounds, the Coordinator must perform root-cause classification; on the third occurrence of the same problem, stop the mechanical loop and escalate to implementation-capability, design-ambiguity, evidence-defect, or candidate-binding handling;
- Role prompts and allowed states follow `07`.

## 10. Context L1 and Context L2

- Whether Context L2 is needed is executed entirely per `06`'s risk-trigger rule;
- Lightweight tasks and single-executor local tasks usually use a task brief and do not build an independent Context L2;
- **Context L2 generation trigger (consistent with `09` §3.2)**: when `ContextL2Policy=Required`, Hermes first enters task state `ContextGenerationPending` before dispatching the Implementer, dispatches the Doc/Design Reviewer to generate Context L2 (must be after its dependencies are satisfied), and only after its completion signal is written back to the ledger does it advance to `Ready` and dispatch the Implementer; stagnation detection and escalation when `ContextGenerationPending` lacks a completion signal: see `09` §11.1;
- Context L1 is updated only when stable architecture, global conventions, core models, or authoritative entry points change substantially;
- At iteration close, check whether Context L1 is affected; if no change, only record the checklist conclusion, do not create an independent `NoChange` development task or Review;
- If there is a change, a formal documentation task must be created and its accuracy independently reviewed; that task must either complete as an iteration-close blocking item or be explicitly approved by the project owner as a follow-up task with ID, owner, and completion time—do not merely record `Changed` and silently close the iteration.

## 11. Task Brief Template

```markdown
### <TASK-ID>: <Name>

> Route: Lightweight / Standard / HighRisk
> Risk: Low / Medium / High
> TaskType: CodingTask / ReworkTask / IterationMergeTask / IntegrationValidationTask / MainMergeTask / DocumentationTask
> TaskState: <only the 09 §3.2 enumeration: Planned / ContextGenerationPending / Ready / InProgress / Submitted / InReview / ChangesRequested / ReworkInProgress / TaskAccepted / MergePending / Integrated / IntegrationVerified; bypass: Blocked / Cancelled>
> CarrierStatus: <only the 09 §3.1 enumeration: NotCreated / Provisioning / Running / Idle / NeedsAttention / Completed / Unavailable / Cancelled>
> VerificationStatus: Verified / VerifiedWithWaivers / Failed / NotRun / NotIssued (filled only by verification-class tasks)
> InvocationID: <generated at dispatch; may be N/A at design time>
> ExpectedExecutionKind: <determined by the current version of the carrier policy artifact; see 09 §12.4>
> PolicyVersion: <recorded at dispatch>
> PolicyArtifactDigest: <recorded at dispatch>
> TaskBranch:
> WorktreePath:
> CodeBaseSHA:
> RequirementsBaselineRef:
> DesignBaselineRef:
> TaskDocumentBaselineRef:
> ContextBaselineRef:
> ReviewedCommitRange:
> ReviewedCommitSet:
> IntegrationMethod: fast-forward / merge-ff-only / exact-cherry-pick / approved-squash
> MergeTarget:
> Dependencies: <TaskID + DependencyType + RequiredState>
> RequirementsAndDesign:
> ContextL2: <path / N/A + reason>

**Goal and Scope**

**Allowed Changes and Must-Investigate**

**Prohibitions**

**Implementation and Boundaries**

**Level 0–3 Responsibility and NotRun**

**External Side-Effect Risk and Safe Validation Method**

**SOLID Review Focus**

**Duration and Critical Path**

**Completion Evidence**
```

## 12. Minimal Whole-Document Structure

```markdown
# <Feature Name> Detailed Development Task

> Document status: Draft
> Version: V0.1
> Requirements and design baseline:
> DevelopmentRoute:
> Project execution configuration:
> Unified state-exchange area and recovery configuration:

## 1. Scope and Track Selection
## 2. Task Graph, Dependencies, and CI Points
## 3. Scheduling Control Plane, Immutable Baseline, and Canary
## 4. Cross-Task System Integration Responsibility
## 5. Detailed Tasks
## 6. Context L1 Impact Judgment
## 7. Level 1–3 Handoff and NotRun
## 8. Confirmed Decisions and Pending Items
```

## 13. Review Checklist

- [ ] Development track is commensurate with actual risk;
- [ ] Task goal, dependencies, and authorization boundary are clear;
- [ ] Coding, rework, iteration merge, integration validation, and main-branch merge tasks are distinguished;
- [ ] Coding tasks have a unique task branch, worktree, CodeBaseSHA, ReviewedCommitRange/ReviewedCommitSet, and MergeTarget;
- [ ] Code, requirements, design, task, and Context baselines are all unambiguous, not referencing another mutable workspace as the sole source of truth;
- [ ] High-risk scheduling has configured IterationID, CanonicalTaskDocumentPath, LedgerLocation ledger Schema, protocol version, recovery policy, and Canary;
- [ ] Each dispatch binds InvocationID, execution mechanism, and idempotent DispatchKey at runtime;
- [ ] Dependency type and required state are explicit; no reuse of the same task ID for two verifications before/after;
- [ ] Continuous-integration point and formal Level 1 are both arranged;
- [ ] Critical cross-task paths have integration responsibility, or explicit `N/A`;
- [ ] Context L2 is risk-triggered, not forced for low-value tasks;
- [ ] Context L1 is updated only when stable facts change;
- [ ] SOLID remains a hard gate, evidence depth commensurate with risk;
- [ ] Role, verification, Review, and merge rules reference the sole source of truth;
- [ ] Unique `CanonicalTaskDocumentPath`, Hermes ledger (LedgerLocation), version, and idempotent write are configured;
- [ ] Hermes ledger is available; when missing, handled per recovery protocol; specified execution mechanism will not be substituted without authorization;
- [ ] External side-effect and no-side-effect verification paths are clear;
- [ ] Duration includes Review, rework, integration, evidence, and manual gate;
- [ ] Pending items are zero.

## 14. Definition of Done

Development tasks may start only when the track selection is reasonable, task boundaries and dependencies are executable, system junctions have an owner, the Context strategy is commensurate with risk, SOLID and verification responsibility are clear, and the applicable development-task document state surface, immutable baseline, recovery policy, and Canary are ready, and the project owner has completed necessary review.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.4 | 2026-08-15 | — | Introduced shared JSON state-exchange area (Codex thread scheduling) |
| V2.5 | 2026-08-20 | WorkBuddy | §4 removed shared-JSON trio, added LedgerLocation/DispatchMode; §8 rewritten to Hermes-ledger state surface; §9 removed shared-JSON polling/Codex subtask; §11 four-tier enum; §13 review checklist removed short-term lock/Codex |
| V2.5 (pending review) | 2026-08-20 | Hermes | §5.2 added dependency-graph cycle detection (reject registration and escalate Richy); §10 added Context L2 generation trigger (when ContextL2Policy=Required, first dispatch Doc/Design Reviewer to generate L2, consistent with 09 §3.2) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
| V2.5 errata | 2026-08-21 | WorkBuddy | Synced source errata bd6a71f: heading-level, wording, and reconciliation-terminology fixes |
| V2.5 errata 2 | 2026-08-22 | Hermes | §8 state table `Integrated` definition adds "and push succeeded", per 08 §3.4 (the current iteration branch is authoritative; merging into a local-only integration branch must not count as Integrated) |
| V3.0-draft | 2026-08-24 | Hermes | V3.0 revision (proposal v5): §11 task-brief template changed `ExpectedExecutionKind` from a static enumeration (WorkBuddy/Codex/Human) to "determined by the current version of the carrier policy artifact" (ref 09 §12.4), and added PolicyVersion and PolicyArtifactDigest as mandatory dispatch fields—eliminating the multi-source-of-truth conflict between the static enum and the dynamic carrier policy (V5 conflictsWithDocs zero-residue item) |

| V3.0-draft-2 | 2026-08-24 | Tiffany-Dev | Three-field template state-domain strict-partition fix: TaskState uses only the 09 §3.2 enumeration (removing the misplaced NeedsAttention/Invalidated—the former belongs to CarrierStatus, the latter is not a TaskState); CarrierStatus uses only the eight states of 09 §3.1 (NeedsAttention homed); VerificationStatus fixed to five values with no cross-domain items. Eliminates state-domain mixing that left the scheduler unable to judge "re-dispatch / await verification / escalate to human" |

| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds (proposal v1-v5 plus nine body rounds) closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
