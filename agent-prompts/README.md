# Sub-Agent Prompt Template Index

> Template version: V2.5
> Template status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)
> Upstream spec: [07-Sub-Agent Delegation & Prompt Spec](../specs/07-subagent-delegation-prompts.md)
> Scheduling protocol: [09-Hermes Ledger & Runtime Spec](../specs/09-hermes-ledger-runtime.md)
> Revision history: see end

## Usage

Each dispatch uses:

```text
00-Common Base
  + one on-demand role template
  + the baselines that role needs
  + Context L2 (when triggered)
  + current Invocation and dynamic data
```

Angle-bracket fields must be replaced. Templates must not be dispatched blank, nor substitute for the executor's autonomous exploration.

All async tasks must first fill `ProtocolVersion`, `IterationID`, `TaskID`, `InvocationID`, `ParentCoordinatorRef` (Hermes identity), and `ExpectedExecutionKind`. The execution mechanism designated by the carrier policy artifact must not be substituted without authorization; no pseudo-independent self-review; only when the project owner pre-approves `ApprovedEquivalent` is an equivalent mechanism allowed.

## Templates

| File | Role |
|---|---|
| [00-Common Base](./00-common-base.md) | Constraints shared by all roles (Hermes injection protocol) |
| [01-Development Agent](./01-development-agent.md) | First-time implementation |
| [02-Coding Reviewer](./02-coding-reviewer.md) | Level 0 independent code review |
| [03-Rework Agent](./03-rework-agent.md) | Close Review findings |
| [04-Test Verification Agent](./04-test-verification-agent.md) | Test execution and evidence classification |
| [05-System Integration Reviewer](./05-system-integration-reviewer.md) | Level 1 whole-system review |
| [06-Final Merge Review Agent](./06-final-merge-review-agent.md) | Frozen-candidate merge eligibility |
| [07-Coordinator Agent](./07-coordinator-agent.md) | Hermes resident dispatch, ledger, and event/cron reconciliation |
| [08-Requirements Review Agent](./08-requirements-review-agent.md) | Requirement scope, criteria, and graded-acceptance review |
| [09-Design Review Agent](./09-design-review-agent.md) | Design coverage, runtime risk, and test-strategy review |
| [10-Development Task Review Agent](./10-development-task-review-agent.md) | Task split, context, scheduling, and gate review |
| [11-Iteration Integration Agent](./11-iteration-integration-agent.md) | Merge precise reviewed task commits into the iteration branch |
| [12-Main-Branch Merge Agent](./12-main-branch-merge-agent.md) | Execute main-branch merge after final gate and authorization |
| [13-Coordinator Periodic Reconciliation](./13-coordinator-periodic-reconciliation.md) | Hermes cron reconciliation: ledger consumption and idempotent health check |
| [14-Coordinator Cold Recovery](./14-coordinator-cold-recovery.md) | Hermes restart recovery: rebuild max-safe state from ledger + Git |
| [15-Scheduling-Protocol Canary](./15-scheduling-protocol-canary.md) | Accept the scheduling control plane in a no-business-side-effect scenario |

The number of role templates does not mean the same number of agents must be launched every iteration. Roles are consolidated into 6 core roles per README §4; the templates below are on-demand role contracts; see the mapping notes in `../specs/07-subagent-delegation-prompts.md` §2 for specific role attribution.

## Dynamic-Data Minimum Requirements

- Repository and workspace;
- ProtocolVersion, IterationID, TaskID, InvocationID, ParentCoordinatorRef, ExpectedExecutionKind, and model;
- Source and target branches;
- TaskType, task branch, worktree, CodeBaseSHA, HeadSHA, ReviewedCommitRange, ReviewedCommitSet, and MergeTarget;
- RequirementsBaselineRef, DesignBaselineRef, TaskDocumentBaselineRef, and applicable ContextBaselineRef;
- Unambiguous baseline or candidate target;
- Upstream document and version;
- Context-package path and state (when applicable);
- Current task, risk, and permission;
- Tests, Findings, or NotRun;
- Expected structured output.
- All async roles have `LedgerLocation`, `CanonicalTaskDocumentPath`, `StateRecordID`, and the ledger version at dispatch injected by Hermes; the execution agent does not write the ledger directly, but outputs a structured protocol header through its carrier channel, which Hermes consumes event-driven + cron-fallback and writes idempotently into the ledger (lock-free). The Coordinator / reconciliation must also provide pause state, recovery strategy, and stop/notify conditions.

---

## Revision History

| Version | Date | Reviser | Note |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex-thread scheduling base (SharedRuntimeStatePath/StateUpdateToolPath/short-lived lock/Codex-subtask fallback ban) |
| V2.5 | 2026-08-20 | WorkBuddy | Whole-doc raised to V2.5: document header, scheduling-protocol reference changed to 09-hermes-ledger-runtime, removed Codex fallback ban to "designated execution mechanism non-substitutable without authorization", shared-JSON triple changed to Hermes-ledger injection description, template-table 07/13/14 descriptions synced |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
