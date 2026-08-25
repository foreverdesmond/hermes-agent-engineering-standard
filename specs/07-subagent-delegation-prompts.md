# Sub-Agent Delegation & Prompt Spec

> Spec version: V3.0
> Document status: Approved (V2.5 is the previous baseline; finalized 2026-08-24, Richy final review passed)
> Applicability: using agents to perform development, Review, testing, scheduling, or merge review
> Author: Tiffany-Dev (V3.0 revision; initial version = WorkBuddy)
> Revised: 2026-08-24
> Reviewer: Richy (approved)

## 1. Single Responsibility

This document only specifies how to choose the execution mechanism and role, assemble prompts, and keep roles independent. The detailed execution contract of each role uses `agent-prompts/README.md` as the sole source of truth (not repeated in the body); task identity, events, ledger, reconciliation, and recovery use `09-hermes-ledger-runtime.md` as the sole source of truth.

The spec defines capability; it does not bind model, vendor, or version. The specific execution instance is configured by the project.

## 2. Select Roles on Demand

V2.5 consolidates into 6 core roles (see README §4); each task enables 2–N as needed:

| Role | Core responsibility | Key boundary |
|---|---|---|
| Coordinator (total scheduling) | Dispatch, periodic reconciliation, gate judgment | Carried by the Hermes resident service; does not do sub-agent hands-on work |
| Implementer (development) | First-time development + rework + L0 unit tests | Only on own feature branch; does not approve own work |
| Reviewer (review) | Review development output (diff/commit) | Review on diff + L0 evidence; Code Immutability Constraint (09 §7.3): under danger-full-access, review in an isolated detached verification workspace; do not modify business source, do not commit candidates, do not merge |
| Integrator (integration) | Merge feature → iteration | Only merges precisely approved commits; does not review own merge |
| Validator (test verification) | Full test after merge (L1 + regression) | danger-full-access; runs the full suite on a clean detached checkout; subject to the Code Immutability Constraint, no merge |
| Doc/Design Reviewer (documentation/design review) | Document review + design + work-package/context | Pure documentation belongs to it; low-risk design not reviewed, high-risk reviewed by another of same role |

**Consolidation sources (V2.4 → V2.5)**: Rework Implementer → Implementer (rework is a sequel to development); System Reviewer → Reviewer (distinguished by Level); Merge Reviewer → Reviewer (merge-eligibility review perspective); Iteration Integrator / Main Merge Executor → Integrator (both are merge execution, only target branch differs); Requirements/Design/Task Reviewer → Doc/Design Reviewer (merged).

**Hard floor retained**: development separated from review; merge execution separated from review; final approval never self-attested by the developer. Lightweight tasks usually need only Implementer + an independent Reviewer.

### 2.1 Execution Mechanism Must Be Explicitly Configured

Role and execution mechanism are two different concepts. The project must record for each dispatch:

```text
ExpectedExecutionKind: <determined by the current version of the carrier policy artifact>
ExpectedModel
Role
InvocationID
PolicyVersion + PolicyArtifactDigest
```

- `ExpectedExecutionKind` is no longer a static enumeration: **available carriers and allocation rules are defined solely by the current version of the "carrier policy artifact"** (see 09 §12.4); specify explicitly at dispatch and validate through the fail-closed pre-dispatch gate; Hermes dispatches per `09-hermes-ledger-runtime.md` and binds `ExecutionRef`;
- Each dispatch's ledger record attaches PolicyVersion and PolicyArtifactDigest, for afterwards auditing "why that carrier was chosen at the time";
- A dispatch returning a temporary request identifier means `Provisioning`, not failure;
- On dispatch failure or carrier unavailable, register `ControlPlaneError` and stop; do not substitute another mechanism without authorization (no pseudo-independent self-review);
- Only when the project owner pre-approves `ApprovedEquivalent` is an equivalent mechanism allowed;
- Role independence cannot be proven merely by a different name; it must have a different `InvocationID` and execution instance.

## 3. Prompt Layering

A single shared development-task base for all roles is no longer required. Use by role:

### 3.1 Truly Common Base

Contains only:

- Protocol version, IterationID, TaskID, InvocationID, parent-coordinator identity, and execution mechanism;
- Repository, workspace, and review target;
- `CodeBaseSHA` and the applicable immutable baselines of requirements, design, task, and Context;
- Protection of the user's existing changes;
- External and destructive-operation authorization;
- Evidence authenticity and `NotRun`;
- Role-allowed states and stop conditions.

### 3.2 Development-Role Increment

- Approved requirements, design, and task brief;
- Context L2 (only when applicable);
- Modification responsibility, must-investigate scope, and completion evidence;
- Autonomous exploration and SOLID hard gate.
- Unique task branch, worktree, `CodeBaseSHA`, the scope allowed to commit, and the target branch prohibited from merging into.
- Structured `DevelopmentSubmission` event header and the Invocation identity that must be returned.

### 3.3 Review-Role Increment

- Requirements and design baseline;
- Unambiguous review target and actual diff;
- Reviewer forms an independent risk model first, then reads the development context package;
- SOLID, test proof boundary, and independent risk assumptions;
- Must not directly modify code or update final task state unless an explicit role change is authorized.
- `TaskBranch`, `CodeBaseSHA`, `HeadSHA`, and precise `CodeBaseSHA..HeadSHA`; Review must not change the current main workspace or other branches.
- Structured `CodingReviewResult` event header separating `ExecutionStatus` and `Verdict`.

### 3.4 Candidate-Version-Role Increment

- Candidate commit / PR revision / stable snapshot;
- Level evidence, NotRun, branch strategy, and target gate;
- No single-task ID or Context L2 required.

### 3.5 Merge-Execution-Role Increment

- Source branch, target branch, precise candidate SHA, valid Review conclusion, and authorization record;
- Allowed Git actions, conflict-handling boundary, post-merge evidence, and prohibition on smuggling in development changes;
- `Iteration Integrator` and `Main Merge Executor` must use a permission contract different from ordinary development tasks.

## 4. Prompt Assembly

```text
Common base (Hermes injection protocol)
  + one role template
  + the formal baselines that role needs
  + optional Context L2
  + this round's dynamic data (Invocation, diff, Findings, tests, or candidate evidence)
```

Dynamic data (Invocation, ledger fields, diff, Findings, tests, or candidate evidence) is injected by Hermes at dispatch; the execution agent does not assemble baselines itself.

Do not send only "complete the task per the document," nor indiscriminately dump the entire document directory.

## 5. Role Independence

- Development and Review use different execution instances or an explicit independent reviewer;
- The Reviewer does not take the developer's summary as fact, nor is it first constrained by the risk scope the developer recommended;
- The same model may take different tasks, but cannot directly approve its own implementation within the same task;
- High-risk final Review should use the strongest independent reasoning the project currently has;
- When an execution instance is replaced, record the recovery point and materials needing re-read.
- Instance replacement must generate a new `InvocationID`, and the old instance is marked invalid or cancelled; two instances must not simultaneously be the valid source of truth for the same stage.
- Coding Review must create an independent task or independent execution instance; do not send "please review" back to the original development task;
- Merge Reviewer only reviews merge eligibility; Merge Executor only executes the already-approved precise merge; review and execution are separated by default.
- Carrier separation (WorkBuddy writes / Codex reviews) does not equal perspective independence; different carriers may still share upstream context and judgment, so an independent review target and risk model must be retained—different carriers alone are not treated as independent review.

### 5.1 Git Permissions

- Implementer and Rework Implementer may only create commits on their own task branch;
- Ordinary development roles must not directly commit or merge into the iteration development branch, long-term integration branch, stable branch, or main branch;
- Reviewer and Validator are subject to the **Code Immutability Constraint**: unified use of danger-full-access to obtain build and test capability, but they must not modify tracked business source, must not commit candidates, and must not merge; Review must run in an isolated detached verification workspace based on the precise candidate commit, with before-and-after HEAD/tree two-way reconciliation—any change invalidates the conclusion (see 09 §7.3);
- The Coordinator is subject to the scheduling-authority Epoch constraint and the pre-dispatch gate constraint (09 §12.3/§12.4);
- Iteration Integrator may only merge specified commits that are already `TaskAccepted` into the iteration development branch;
- Main Merge Executor may only merge the precise candidate into the main branch after `MergeApproved` and explicit project-owner authorization;
- When a conflict requiring implementation change occurs, the merge executor stops and returns to the development/rework loop.

## 6. Reviewer's Risk Counter-Evidence

The Reviewer must form an independent risk hypothesis.

- When an executable and important risk exists, verify at least one counter-example the developer did not cover;
- When no worthwhile extra counter-example exists, state the basis for the judgment;
- Do not manufacture meaningless tests to meet a quantity quota;
- Counter-examples cannot replace the full review of requirements, design, and SOLID.

## 7. SOLID Role Responsibility

- Implementer states the SOLID impact of this change at commit time;
- Coding Reviewer independently completes S/O/L/I/D judgment;
- System Reviewer checks whether responsibility leakage, wrong abstraction, or dependency-inversion violation appears after task combination;
- Low-risk tasks may be recorded concisely; high-risk or trade-off cases must be expanded;
- SOLID is always a hard gate; it is not cancelled by omitting Context L2 or simplifying the development track.

## 8. Coordinator Minimum Rules

When using the Coordinator:

- One task keeps only one valid development instance at the same stage;
- State is judged by actual result, review target, and diff;
- After development completes, enter independent Review; on failure return to the original task for rework;
- On Review pass, first form `TaskAccepted`; then dispatch an independent iteration-integration task to form `Integrated`, then execute affected integration verification to form `IntegrationVerified`;
- Dispatch downstream only when dependencies are satisfied; a dependency needing upstream implementation defaults to waiting for `Integrated`; the dependency graph is recorded by the Hermes ledger, and Hermes actively triggers downstream when dependencies are satisfied;
- After dispatch, Hermes consumes the ledger via event sources + scheduled reconciliation fallback; the execution agent reports via carrier channel, and Hermes writes `PendingConsumption` and processes immediately;
- Reconciliation only processes state changes; it does not resend the same instruction;
- Active heartbeats and long logs may be kept in the derived evidence/audit cache; task state, Invocation, execution reference, pending-consumption signal, and consumption confirmation must be written to the Hermes ledger;
- Runtime state is written to the ledger by Hermes per `09-hermes-ledger-runtime.md`; when the ledger is unavailable, follow the recovery protocol;
- Each dispatch has `InvocationID`, `DispatchKey`, immutable code/document baseline, and causal event;
- Creation request, formal task, thread state, task state, and evidence state are registered separately;
- All state signals are read at-least-once by `RecordID + SignalRevision` and consumed idempotently;
- When the project pauses, stop new side effects; on resume, process unconsumed events first;
- The Coordinator does not approve its own implementation.

The Coordinator maintains at least the following runtime mapping; do not infer merely from UI labels:

```text
RecordID, TaskID, TaskType, InvocationID, ExpectedExecutionKind, ExpectedModel,
ExecutionRef, TaskBranch, WorktreePath,
CodeBaseSHA, RequirementsBaselineRef, DesignBaselineRef,
TaskDocumentBaselineRef, HeadSHA, ImplementerTask, ReviewerTask,
CarrierStatus, TaskState, EvidenceState, SignalRevision, SignalState,
ProducedAt, ConsumedAt, LastEventFingerprint
```

### 8.1 Pre-Dispatch Mandatory Checklist

Before dispatching any task, the following must be checked:

- TaskType, role template, and allowed output states;
- Dependency type and required state;
- Task branch, worktree, CodeBaseSHA, MergeTarget, and candidate form;
- InvocationID, execution mechanism, expected model, parent-coordinator identity, and idempotent DispatchKey;
- Immutable baselines of requirements, design, task, and Context;
- File scope and whether it conflicts with other active tasks;
- Whether Context L2 is applicable, exists, and is still valid;
- Git permission, external-side-effect permission, and stop conditions;
- Expected test scope, delivery evidence, and subsequent Reviewer;
- Whether a valid instance already exists at the same stage, to prevent duplicate dispatch.
- The Hermes ledger (LedgerLocation) is writable, single-instance idempotent (DispatchKey dedup), and not currently Paused/recovering;

Dispatch is forbidden when any required item is missing or conflicts are unresolved.

### 8.2 State Production, Consumption, and Reconciliation

Standard scheduling loop:

```text
Hermes registers DispatchKey/Invocation/state record in the ledger and dispatches
→ binds ExecutionRef (unified execution reference)
→ an event source arrives or the scheduled reconciliation fallback triggers
  → PendingConsumption: only read the execution carrier bound by the record, verify delivery, then idempotently dispatch the next action
  → no new state: do not iterate carriers, only do ledger/document/Git health reconciliation
```

Delivery verification includes at least: protocol version, Invocation, final state, `HeadSHA`, expected files, test evidence, out-of-scope changes, and subsequent Review/integration actions. A task completing within the first minute must be processed immediately, not waiting for the seventh minute or another configured cycle.

> **IntegrationVerified execution subject**: after `Integrated`, `IntegrationVerified` is owned by the **Integrator (or an independent `IntegrationValidationTask`)** — it runs/triggers affected integration checks after merging the precise commit, and reports a structured protocol header (`IntegrationCommit` + `IntegrationStatus`) through the carrier channel; Hermes consumes it into the ledger and then triggers downstream Level 1. It does not depend on an external CI webhook (see `09` §6.1 / `08` §3.4).

A UI showing idle/completed, an unchanged Git HEAD, or a temporarily-empty read interface alone cannot prove "no Review/completion state." Task-state discovery uses only the development-task document; complete subtask results provided by the user should be written to the corresponding record's `PendingVerification` and verified immediately, not overridden by an old state.

### 8.3 Interruption and Block Judgment

The following intermediate phenomena alone cannot justify interruption: unfinished code, a single compile failure, a single test failure, long-running tests still producing output, or a worktree continuously changing.

Interruption is allowed only when the executor explicitly reports `Blocked`/`NeedsScopeChange`, an authorization or file-scope conflict appears, the project owner requests a stop, or continuous checks confirm no activity and no recovery path. Before interrupting, the latest message, task state, and available activity evidence must be read.

### 8.4 Review–Rework Loop Escalation

- First-round `ChangesRequested` is normal rework;
- On the second round with the same type of Finding, Hermes must compare the two rounds' conclusions and classify the root cause;
- On the third occurrence of the same problem, stop mechanical dispatch, judge implementation-capability mismatch, requirement/design ambiguity, candidate-binding error, or evidence-rule error, and report the needed ruling to the project owner;
- When code `HeadSHA` is unchanged and only the state area/evidence is updated, do not re-run the full Coding Review or full test suite.

## 9. Prompt Quality Gate

- [ ] Role and allowed output states are explicit;
- [ ] Protocol version, IterationID, TaskID, InvocationID, parent-coordinator identity, and execution mechanism are explicit;
- [ ] Review target is unambiguous;
- [ ] Only the baselines the role truly needs are provided;
- [ ] Context L2 is provided only when triggered;
- [ ] Modification permission and read-only investigation are separated;
- [ ] Task branch, CodeBaseSHA, HeadSHA, ReviewedCommitRange/ReviewedCommitSet, and merge target are explicit;
- [ ] Code, requirements, design, task, and Context baselines are unambiguous;
- [ ] SOLID, test, evidence, and NotRun responsibility are explicit;
- [ ] External-operation boundary is explicit;
- [ ] Review perspective is not pre-limited by the development context;
- [ ] Stop conditions are explicit;
- [ ] The pre-dispatch checklist has passed, and document state production/consumption and periodic reconciliation are configured;
- [ ] A unique ledger (`LedgerLocation`), idempotent `DispatchKey`, CoordinatorEpoch fencing validation, pause gate, and applicable Canary are configured;
- [ ] The execution mechanism designated by the policy artifact is not substituted without authorization (no pseudo-independent self-review);
- [ ] No unnecessary specific model, business, or tool is bound.

## 10. Template Maintenance

Templates may be extended per project, but role goal, permission, output state, and prohibited behavior must not be mixed across roles. If a prompt change affects a running task, send the complete updated role contract rather than scattered supplements that cause conflict.

## 11. Definition of Done

Agent delegation may start only when the project has selected the minimal necessary roles, filled correct baselines and dynamic data for each role, kept development and Review independent, and passed the prompt quality gate.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.4 | 2026-08-15 | — | Introduced CodexThread/SubAgent execution mechanism and shared-JSON polling |
| V2.5 | 2026-08-20 | WorkBuddy | §2.1 four-tier enum (WorkBuddy/Codex/Human/ApprovedEquivalent); §4 dynamic data injected by Hermes; §5 carrier separation ≠ perspective independence; §8 shared JSON → Hermes ledger + event/cron; ExecutionThreadID → ExecutionRef |
| V2.5 | 2026-08-20 | Hermes | Review revision: §2 role list consolidated to 6 core roles, extended roles merged into mapping notes |
| V2.5 (pending review) | 2026-08-20 | Hermes | §8.2 synced IntegrationVerified execution subject = Integrator (or independent IntegrationValidationTask), not depending on external CI webhook (consistent with 09 §6.1 / 08 §3.4) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
| V2.5 errata | 2026-08-21 | WorkBuddy | Synced source errata bd6a71f: heading-level, wording, and reconciliation-terminology fixes |
| V3.0-draft | 2026-08-24 | Hermes | V3.0 revision (proposal v5): ① §2 role-table Reviewer row and §5.1 Git permissions: "read-only" changed to the "Code Immutability Constraint"—unified danger-full-access (build/test requires write permission), isolated detached verification workspace + before-and-after HEAD/tree two-way reconciliation; Coordinator adds Epoch constraint and gate constraint; ② §2.1 `ExpectedExecutionKind` static enumeration changed to reference the current version of the "carrier policy artifact", dispatch records add PolicyVersion+PolicyArtifactDigest |
| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds (proposal v1-v5 plus nine body rounds) closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
