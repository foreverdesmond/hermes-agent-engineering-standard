# Hermes Capability Boundary List

> Spec version: **V2.5 (draft)**
> Document status: **Approved (V2.5 final baseline)**
> Author: Hermes (Tiffany)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)
> Revision history: see end
> Positioning: **P0 prerequisite** of the V2.5 upgrade plan. The three outlines, the review report, and the go-live plan all assume Hermes has certain capabilities; this list empirically confirms and pins down **Hermes's actual capabilities** as the dependency foundation for rewriting `00-common-base`, `09-ledger-spec`, and the dispatch mechanism.

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
| ① Feishu long connection (lark-ws) | Receive **WorkBuddy** reply messages | Passive; messages may be lost (empirically confirmed: WB replied but Hermes did not receive) → needs fallback |
| ② Codex gateway polling `GET /v1/threads/:id` | Check **Codex** thread progress | Actively poll status: inProgress → completed/failed |
| ③ Hermes cron timed polling | **Fallback reconciliation** | Frequency ~1 minute; backfills dropped/lost events |
| ④ TG push | Notify Richy of key nodes | **0 token** (read ledger structured fields + template assembly, no LLM) |

**Reconciliation conclusion**: adopt **event (①②) + cron timed polling (③) dual channel**; event loss does not affect correctness (cron fallback reconciliation).

## 4. Scheduled Scheduling (cron) Capability

| Item | Confirmed Result |
|---|---|
| Supported syntax | `30m` / `every 2h` / standard 5-field cron `0 9 * * *` / ISO one-shot timestamp |
| Minimum precision | **Minute-level** (5-field cron, no second-level) |
| Can it carry scheduling reconciliation | **Yes**. cron reconciles once per minute, handling pending records and health checks, consistent with "event + cron fallback" |

## 5. Agent Dispatch Capability (Hermes → each carrier)

| Carrier | Dispatch method | Sync? | Reconciliation method | Git permission |
|---|---|---|---|---|
| **WorkBuddy** | Feishu post message @WB | Async (one-way delivery) | Passively wait for WB interactive reply + cron fallback | ✅ commit allowed by default |
| **Codex** | `POST /v1/threads` (gateway) | Sync (wait) / Async | Actively poll `/threads/:id` | ⚠️ requires `sandbox:danger-full-access` |

**Dispatch parameters** (Codex gateway):
```text
POST http://10.192.241.5:4501/v1/threads
body: { prompt, cwd, model, modelProvider, sandbox, approval }
- modelProvider: openai (native model) / opencodex (CodexSplit third-party)
- sandbox: read-only / workspace-write (write files) / danger-full-access (when git is needed)
```

## 6. Tool Permission Tiers (sandbox policy assigned to agents)

| Task type | sandbox | Description |
|---|---|---|
| Read-only research / Review | read-only | Reviewer, review |
| Ordinary file writing | workspace-write | Writing documents / evidence |
| **Requires git commit** (development / integration / design) | **danger-full-access** | `.git` is a protected path under `workspace-write` and will block git |

**Empirical basis** (verified 2026-08-19):
- Under `workspace-write`, `git add` reports `index.lock: Operation not permitted` (`.git` blocked by sandbox)
- Under `danger-full-access`, `git status / worktree add / add / commit / push / reset` all **succeed**
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
| Push | TG push of **key nodes + 0 token + ledger link**: Submitted / Approved / ChangesRequested / Integrated / needs-authorization / abnormal-failure |
| Link | Ledger entry path; Richy opens on demand, full volume not proactively dumped |

## 9. Known Limitations (honest disclosure)

1. **Feishu push may be lost** (empirically confirmed): must rely on cron polling as fallback, cannot depend on events alone.
2. **Codex partial-model output occasionally anomalies**: deepseek-flash short tasks occasionally end early / stream interrupted returning empty; pro is more stable; the gateway extracts the final answer from the last agentMessage.
3. **GPT native models consume Codex session traffic**; quota exhaustion reports usageLimitExceeded (resets around the 20th of each month).
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
