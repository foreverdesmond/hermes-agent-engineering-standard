# Hermes Capability Boundary List

> Spec version: **V3.0**
> Document status: **Approved (finalized 2026-08-24)**
> Author: Hermes (Tiffany)
> Created: 2026-08-20
> Last updated: 2026-08-24
> Reviewer: Richy (approved)
> Revision history: see end
> Positioning: **a spec-side summary of non-normative instance capability registrations**—deployment facts (endpoints/IPs/commands/phenomena) have moved to the instance registry (runtime ledger directory); this file keeps only abstract conclusions. Historical positioning: the **P0 prerequisite** of the V2.5 upgrade plan. The three outlines, the review report, and the go-live plan all assume Hermes has certain capabilities; this list empirically confirms and pins down **Hermes's actual capabilities** as the dependency foundation for rewriting `00-common-base`, `09-ledger-spec`, and the dispatch mechanism.

---

## 1. Hermes Runtime Form

| Item | Confirmed Result |
|---|---|
| Deployment form | **Single-machine, single-instance** (one server runs one Hermes, producing only one scheduler instance) |
| Inference | No multi-Coordinator contention → **no coordination-lease lock needed**; idempotency guaranteed by `DispatchKey` deduplication |
| Residency | **Resident service** (not a temporary thread), running continuously, naturally acting as the "Coordinator" |

## 2. State Persistence / Ledger Carrier

| Item | Confirmed Result |
|---|---|
| Own storage | The Hermes framework uses a **SQLite session library** to store sessions / memory (framework-level, not the scheduling ledger) |
| Project ledger carrier | **Local JSON state file** (within the project directory, **not in Git**) + optional SQLite; maintained by Hermes |
| Ledger contents | Task / stage / status / evidence pointer / dependency graph / scheduling consumption watermark (ConsumedRevision) |
| Immutable snapshot | The `TASK-STATE-EXCHANGE` block of the development task document remains the **Git persistent snapshot** (for restart / disaster recovery) |
| Durable recoverability | Ledger is persisted; Hermes can read the ledger + Git to rebuild after restart |

> **Important distinction**: **Scheduling ledger ≠ Hermes session library**. The Hermes session library is the framework's SQLite for storing chat / memory; the scheduling ledger is the local JSON state file Hermes maintains specifically for development iterations (scheduling-only, containing task/stage/evidence/dependency). They are different stores and must not be mixed.

## 3. Event Source and Periodic Reconciliation Capability

| Event / Channel | Coverage | Description |
|---|---|---|
| ① Push event source | Receive **interactive-carrier** reply messages | Passive; messages may be lost (empirically confirmed) → scheduled reconciliation fallback is mandatory; do not depend on events alone; implementation registered in the instance record |
| ② Polling event source | Check **async-carrier** execution progress | Actively poll status until completed/failed; implementation registered in the instance record |
| ③ Scheduled reconciliation fallback | **Fallback reconciliation** | Frequency meets the agreed time limit (registered in the instance capability record); backfills dropped/lost events |
| ④ Key-node notification output | Notify Richy of key nodes | Structured template assembly, no LLM (channel implementation registered in the instance capability record) |

**Reconciliation conclusion**: adopt **event sources (①②) + scheduled reconciliation fallback (③) dual channel**; event loss does not affect correctness.

## 4. Scheduled Scheduling (cron) Capability

| Item | Confirmed Result |
|---|---|
| Supported syntax | Period intervals / standard schedule expressions / one-shot timestamps (concrete syntax registered in the instance capability record) |
| Minimum precision | **Minute-level** (no second-level) |
| Can it carry scheduling reconciliation | **Yes**. Scheduled reconciliation meets the agreed time limit, handling pending records and health checks, consistent with "events + fallback" |

## 5. Agent Dispatch Capability (V3.0: abstract conclusions)

- Each execution carrier connects through a **dispatch adapter**; the adapter contract (dispatch / identity binding / sync-async results / failure semantics) is in 09 §7.1;
- The available-carrier set, channel implementations, and endpoint configuration of the current deployment are decided by the "carrier policy artifact" and the instance capability registry; this list does not record concrete endpoints or vendor parameters;
- Execution-carrier adapters support synchronous or asynchronous dispatch and can return a traceable execution identity (ExecutionRef).

## 6. Tool Permission Tiers (sandbox policy assigned to agents)

| Task type | sandbox | Description |
|---|---|---|
| ~~Read-only research/Review~~ → V3.0 unified danger-full-access | Reviewer/Validator subject to the Code Immutability Constraint (may build and test, must not modify business source), working in isolated detached verification workspaces |
| Ordinary file writing | workspace-write | Writing documents/evidence |
| **Requires git commit** (development/integration/design) | **danger-full-access** | `.git` is a protected path under workspace-write and blocks git |

**Empirical basis** (verified 2026-08-19):
- Sandboxes may reject Git metadata writes (concrete error signature registered in the instance record)—V3.0 has unified on danger-full-access to avoid it
- Git operations under danger-full-access have passed permission verification (including worktree management and commit/push; the tested command list is registered in the instance record)
- Custom Permission Profiles are constrained by host requirements in a managed environment and are unreliable

## 7. Restart Recovery Semantics (A4, three tiers)

| Tier | Trigger | Hermes action | Failure handling |
|---|---|---|---|
| Hot recovery | A single tool / call occasional anomaly | Log, auto-resume from most recent ledger snapshot; retry | Still failing → wait for cron reconciliation fallback |
| Cold-Start Recovery | Process crash / restart | **Fully automatic**: read ledger + Git to rebuild `ConsumedRevision` → RecoveryOnly validation → **auto-completes on validation pass (no Richy intervention needed)** | ❌ **Failure auto-escalates to disaster level** (only disaster level needs Richy) |
| Disaster recovery | Ledger lost/corrupted (or cold recovery failed) | Rebuild from Git task-document snapshot, determine provable state, downgrade the rest for later verification | **Requires Richy's intervention and approval** |

## 8. Notification and Presentation

| Item | Confirmed Result |
|---|---|
| Ledger dashboard | **No new development dashboard**; Hermes maintains the ledger as the source of truth |
| Push | The configured notification channel pushes **key nodes + ledger link**: Submitted / Approved / ChangesRequested / Integrated / needs-authorization / abnormal-failure (channel implementation registered in the instance capability record) |
| Link | Ledger entry path; Richy opens on demand, full volume not proactively dumped |

## 9. Known Limitations (honest disclosure)

1. **Push event sources may lose messages** (empirically confirmed): must rely on polling/scheduled reconciliation fallback; cannot depend on events alone.
2. **Some execution models have occasional output anomalies** (short tasks ending early / stream interrupted returning empty); the adapter layer extracts the final answer from the last valid output. Affected models and mitigations are registered in the instance capability record.
3. Execution carriers have **quota and rate limits**; when exhausted they report quota errors and enter the carrier-unavailable flow; concrete cycles and error signatures registered in the instance record.
4. **Hermes does not perform git on their behalf**: worktree creation / commit / merge is handled by the respective role agent itself using danger-full-access; Hermes only dispatches parameters, runs periodic reconciliation, and judges gates.
5. **Multi-task parallelism needs constraints**: tasks with conflicts are not set to parallel; genuine conflicts require Richy's coordination (see `Hermes Process & Boundary Resolution` C2).

---

*This list is the empirical confirmation of Hermes's capabilities and is the dependency foundation for rewriting `00-common-base`, `09-ledger-spec`, and the dispatch mechanism. Gaps are filled during implementation.*

---

## Revision History

| Version | Date | Reviser | Description |
|---|---|---|---|
| V2.5 (draft) | 2026-08-20 | Hermes | First release: confirmed Hermes runtime form / ledger / events / cron / dispatch / permissions / recovery capabilities; added document version number and review-status meta info (per Richy feedback) |
| V2.5 (draft) | 2026-08-20 | Hermes | Review revision: fixed typo 'native model'; §7 cold recovery synced with 09 (auto-complete, no Richy needed); §2 explicitly distinguished scheduling ledger from Hermes session library |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as the official V2.5 baseline |


---

## V3.0 Appendix (revised by Tiffany-Dev, 2026-08-24): Sandbox and Privileged-Operation Boundary Updates

1. **All Codex dispatches uniformly use `danger-full-access`** (incl. Review/Validator): the V2.5 read-only sandbox was empirically unable to compile and run tests; the permission expansion is offset by the "Code Immutability Constraint" (09 §7.3).
2. **Cross-profile service isolation iron rule**: no agent may stop/restart/edit another profile's services; troubleshooting uses non-invasive means; the only legal path for cross-profile coordination is escalation to Richy.
3. **A security rejection = a permission signal**: an operation rejected by a guardrail must be handed to Richy to execute; retrying via scheduled tasks, background processes, or any other bypass is prohibited (historical incident tools listed in the instance record).
4. **Privileged operations leave audit trails**: service or scheduled-task management operations etc. require Richy's authorization first and an OPS-AUDIT record (tool list in the instance record).
   (Origin: the 2026-08-22 gateway privilege-escalation incident; incident archive location registered in the instance record)
5. The "privileged operations governance" shared skill (name in the instance record) is a mandatory principle across all profiles.

| V3.0-draft | 2026-08-24 | Tiffany-Dev | Deployment form changed to multi-driver + CoordinatorEpoch FencingToken (original single-machine single-instance corollary voided); unified danger-full-access (incl. Review, Code Immutability Constraint); added privileged-operation boundaries (cross-profile isolation / rejection-to-human / OPS-AUDIT). See the V3.0 appendix at the end |
| V3.0 errata | 2026-08-25 | Hermes | WB review F-002: removed residual concrete-carrier dispatch table in §5 (deployment facts belong to the instance registry); §4 reconciliation wording abstracted |
| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
