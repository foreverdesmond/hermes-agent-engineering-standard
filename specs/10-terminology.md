# Terminology

> Spec version: V3.0
> Document status: Approved (finalized 2026-08-24; final review by Richy)
> Scope: All specification documents, prompt templates, and cross-role communication in this repository
> Author: Tiffany-Dev (V3.0 revision; initial version = WorkBuddy)
> Created: 2026-08-21
> Last updated: 2026-08-24
> Reviewer: Richy (approved)
> Revision history: See the end of this document

## 1. Purpose

This document consolidates the fixed terms, abbreviations, and official English terms used by the repository's standards. During translation, review, or authoring of new documents, use the terms defined here as the baseline to prevent terminology drift across documents.

## 2. Roles and Scheduling

| Chinese | English / Original | Handling notes |
|---|---|---|
| 总调度（Hermes） | Coordinator (Hermes) | Hermes is the implementation instance / codename for this role, not a synonym for Coordinator. In prose, use “Coordinator—implemented by Hermes” where needed. |
| 开发 | Implementer | The execution role for initial implementation, rework, and L0 unit tests. |
| 审核 | Reviewer | Independent code, design, or documentation review role. |
| 集成 | Integrator | Execution role that merges reviewed commits into the iteration or main branch. |
| 测试验证 | Validator | Role responsible for full testing and regression verification after merging. |
| 文档 / 设计审核 | Doc/Design Reviewer | Review role for documentation, design, work packages, and context packages. |
| 执行载体 | Execution Carrier | The adapter endpoint that executes a task. The concrete set is determined by the current Carrier Policy Artifact; it is distinct from a Role. |
| 执行机制 | Execution Mechanism | `ExpectedExecutionKind`, determined by the current Carrier Policy Artifact rather than a static enumeration (V3.0). |
| 伪独立自审 | Pseudo-Independent Self-Review | Presenting a review as independent merely by naming a different carrier is prohibited; role independence must be demonstrated by different InvocationIDs and execution instances. |

## 3. Context, Baselines, and Packages

| Chinese | English / Original | Handling notes |
|---|---|---|
| 上下文包（Context L2） | Context Package (Context L2) | A risk-triggered, single-task navigation package; L1 is stable architecture material and L3 is autonomous exploration by the executor. |
| 上下文 L1 | Context L1 | Project-level stable operating instructions; its physical carrier is the repository-root `AGENTS.md`. |
| 不可变基线 | Immutable Baseline | An unambiguous reference to requirements, design, tasks, documents, or context: Git commit/tree, controlled revision, diff hash, or approved snapshot. |
| 代码基线 | CodeBaseSHA | SHA of the commit at which a task starts. |
| 头提交 | HeadSHA | Current commit on a task branch; a new HeadSHA automatically invalidates the old Review as `Superseded`. |
| 审核范围 | ReviewedCommitRange / ReviewedCommitSet | The actual commit range and set reviewed. |

## 4. States, Ledger, and Gates

| Chinese | English / Original | Handling notes |
|---|---|---|
| 台账 | Ledger / Tracking Ledger (Ledger) | The only live source of truth for task runtime state (local JSON + optional SQLite, not committed to Git). Use “Tracking Ledger” where needed to avoid financial-accounting ambiguity. |
| 任务状态 | TaskState | Development workflow state, from `Planned` through `Integrated` and `IntegrationVerified`. |
| 证据状态 | EvidenceState | `Unread` / `Received` / `PendingVerification` / `Validated` / `Rejected` / `Superseded` / `Consumed`. |
| 执行载体状态 | Carrier Status | `NotCreated` / `Provisioning` / `Running` / `Idle` / `NeedsAttention` / `Completed` / `Unavailable` / `Cancelled`. |
| 待消费 | PendingConsumption | A produced signal waiting for Hermes to read and consume idempotently. |
| 已接受任务 | TaskAccepted | Created after an independent Review passes; eligible for iteration integration. |
| 已集成 | Integrated | State after the exact reviewed commit has been merged into the target branch. |
| 集成已验证 | IntegrationVerified | State after the Integrator's affected integration checks pass. |
| 候选冻结 | Candidate Freeze | A human-gated event in which the project owner freezes the iteration candidate and triggers Level 1 dispatch. |
| 合并已批准 | MergeApproved | State after final merge review passes and the project owner explicitly authorizes the merge. |
| 已合并 | Merged | State after main-branch merge execution completes; it does not automatically mean `ReleaseReady`. |
| 冻结 | Frozen / Freeze | A candidate version is locked against further changes; this means controlled locking, not temporary pausing. |

### 4.1 Carrier Policy and Concurrency (V3.0 New)

| Chinese | English / Original | Handling notes |
|---|---|---|
| 载体策略制品 | Carrier Policy Artifact | A versioned data file defining available execution carriers and allocation rules; the sole source of truth for carrier selection. The standard references its contract without copying its contents. |
| 策略不可变摘要 | PolicyArtifactDigest | Cryptographic digest of the Carrier Policy Artifact (such as SHA-256); recorded with PolicyVersion in the runtime ledger for every dispatch to detect same-version tampering. |
| 协调者纪元 | CoordinatorEpoch | The scheduling-authority FencingToken: monotonically increasing, with exactly one current owner at any time; dispatches and consumption by a non-current Epoch are rejected. |
| 原子条件换主 | Atomic Conditional Takeover | Scheduling authority changes only when the current Epoch and StateRevision still match; otherwise takeover fails. |
| 移交标识 | TransferID | Unique identifier for one scheduling-authority handover; a normal handover requires acknowledgements from both old and new owners. |
| 失联恢复超时 | LostOwnerTimeout | Threshold for declaring the old scheduler disconnected (default: two scheduling cycles); after timeout, recovery takeover requires Richy's authorization. |

## 5. Verification Levels and Evidence

| Chinese | English / Original | Handling notes |
|---|---|---|
| 验证层级 0–3 | Verification Level 0–3 | L0 single task, L1 iteration integration, L2 production-shaped system, L3 real external environment. |
| 未运行 | NotRun | An acceptance item not executed because of a missing environment or authorization; record the reason, risk, and waiver approval, and never disguise it as passed. |
| 替身 | Stub / Mock / Fake | A test substitute; declare what it can and cannot prove. |
| 证明缺口 | Proof Gap | A gap in proving real behavior caused by a test substitute or environment limitation. |

## 6. Testing and Quality

| Chinese | English / Original | Handling notes |
|---|---|---|
| SOLID | SOLID | Preserve the acronym; it is a hard gate and may not be removed because of Context L2 or development-route simplification. |
| 反例 | Counter-Example | The Reviewer independently constructs a counter-example not covered by the developer to test the risk assumption. |
| 风险反证 | Risk Counter-Evidence | The Reviewer forms an independent risk hypothesis and attempts to verify it. |

## 7. Scheduling Protocol and Recovery

| Chinese | English / Original | Handling notes |
|---|---|---|
| 幂等派发键 | DispatchKey | De-duplication by `IterationID + TaskID + Stage + TargetIdentity`; scheduling authority uniqueness is enforced by the CoordinatorEpoch FencingToken (V3.0). |
| 调用标识 | InvocationID | Unique identifier for each dispatch; only one valid instance is allowed for the same task and stage. |
| 执行引用 | ExecutionRef | Unified execution-carrier identity; old V2.5 two-part identity fields are mapped to it on read for compatibility. |
| 对账 | Reconciliation | Hermes consumes ledger signals and reconciles state; use this term consistently instead of mixing it with “patrol”. |
| 周期巡检 | Periodic Reconciliation | Scheduled fallback reconciliation loop for the event channel; its frequency is recorded in the instance capability record. |
| 三级恢复 | Three-Tier Recovery | Hot recovery / cold recovery / disaster recovery. |
| 最大安全状态 | Maximum-Safe State | During recovery, restore only to the highest state supported by evidence. |
| 调度协议验收 | Scheduling-Protocol Canary | Acceptance of the control plane in a scenario with no business side effects. |
| 验收失败 | CanaryFailed | State in which the Canary fails; freeze the start of real high-risk iterations until it passes or a clear waiver is recorded. |

## 8. Permissions and Boundaries

| Chinese | English / Original | Handling notes |
|---|---|---|
| 沙箱分级 | Sandbox Tiers | `read-only` / `workspace-write` / `danger-full-access`. |
| 全权访问 | danger-full-access | Used by roles that need to commit or merge; `.git` is a protected path under `workspace-write`. |
| 能力边界 | Capability Boundary | See `Hermes Capability Boundary List`. |
| 流程与边界决议 | Process & Boundary Resolution | See `Hermes Process & Boundary Resolution`. |

### 8.1 Execution Failure and Waivers (V3.0 New)

| Chinese | English / Original | Handling notes |
|---|---|---|
| 执行故障 | ExecutionFailure | `completed` but missing a structured protocol header or a substantive business conclusion; do not advance business state; resend at most once, then `PendingVerification`. |
| 最大自动尝试次数 | MaxAutomaticAttempts | Persistent count by `TaskID+Stage+TargetIdentity`; default and maximum 3; handover or instance replacement does not reset it; reaching the limit sets `NeedsAttention`. |
| 代码不可变约束 | Code Immutability Constraint | Reviewer/Validator may build and test with `danger-full-access`, but may not modify tracked business source, candidate commits, or merge; reconcile HEAD/tree before and after, and any change invalidates the conclusion. |
| 干净检出验证 | Clean-Checkout Verification | L2+ must verify the exact candidate commit in an independent detached worktree; ignored dependencies require a reproducible controlled fallback source. |
| 豁免边界七项组 | Waiver Boundary Tuple | Command/scenario + failure signature + impact scope + risk + substitute evidence + authorizer + expiry candidate identity; missing any item makes it invalid. |
| 豁免通过 | VerifiedWithWaivers | Passing conclusion with a valid waiver; it does not satisfy a gate requiring “no waivers”. `Verified` means fully passed without waivers. |

## 9. Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.5 | 2026-08-21 | WorkBuddy | First terminology table: consolidated fixed terms, abbreviations, and official translations as the cross-document consistency baseline. |
| V2.5 final | 2026-08-21 | WorkBuddy | Reviewed and approved, marked as the official V2.5 baseline. |
| V3.0-draft | 2026-08-24 | Hermes | Added the V3.0 terminology groups for carrier policy and concurrency, and execution failure and waivers. |
| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived. |
