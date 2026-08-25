# V2.5 Hermes Coordinator Mode · Process & Boundary Resolution

> Status: **Archived (historical decision basis; not a normative entry point)**
> Nature: positioned the same as V3.0-proposal.md—records historical discussions and decision basis; not current executable rules;
> Feishu knowledge base, TG, deepseek, etc. mentioned here are historical deployment facts; current rules are governed by the body of `specs/`.
> Date: 2026-08-20
> Participants: Richy + Hermes (Tiffany), continuing historical review discussions and migration design (related research / review / upgrade-plan documents archived to an external knowledge base; location registered in the instance record)
> Note: This document records the **complete process** and **all boundary resolutions** that Richy and Hermes confirmed item by item during the Hermes V2.5 migration. It is the implementation-level refinement of the review report / upgrade plan, and serves as the factual baseline for the subsequent "Hermes Capability Boundary List" and the implementation schedule.

---

## I. Overall Process (Final Version)

```text
[Richy + GPT-5.6-Sol]: Requirements analysis → Boundary partitioning → Detailed design → Task breakdown
        │              (the task list notes each task's carrier agent and model)
        ▼
[Hermes reviews the development tasks: reasonable / executable]──feedback──► [Richy + GPT-5.6-Sol revise]
        ▼
Hermes: create tracking ledger (SQLite) → dispatch per task list
        ▼
Implementer (carrier per list): create worktree → develop → L0 unit tests (record evidence) → commit
        ▼
Reviewer (one-to-one, reads the commit read-only directly inside that worktree)
        ▼  Approved / ChangesRequested (rework → review only the changed parts)
Integrator: merge into iteration → uniformly clean up worktree
        ▼
Validator: L1 + regression (automated)
        ▼  [passed]
════════════ Real-environment gate: Richy authorizes + Richy personally verifies in the real test environment ════════════
        ▼  approved
Release → production/main
```

---

## II. Complete Boundary Resolution Details (Discussion & Confirmation Record)

### A. Capability Carrier (Hermes Capability Boundary)

| # | Resolution Item | Conclusion |
|---|---|---|
| A1 | Hermes tracking-ledger carrier | Decided by Hermes (leaning SQLite); the subsequent "Hermes Capability Boundary List" to be completed independently by Hermes |
| A2 | Event source & periodic reconciliation | **Message-receiving + scheduled polling (≈1 minute) dual channel**; Feishu push may be lost (empirically confirmed: WB replied but Hermes did not receive it), so polling is needed as a fallback |
| A3 | Deployment form | **Single-machine, single-instance** (subject to Hermes confirmation: one server running one Hermes produces only a single instance) |
| A4 | Recovery semantics | **Hot**: a single anomaly auto-resumes from the most recent ledger snapshot; **Cold**: after restart, fully automatically reads ledger + Git to rebuild, escalating to disaster level on failure; **Disaster**: ledger lost/corrupted (or cold recovery failed), rebuild from Git task-document snapshot, requires Richy's intervention and approval |
| A5 | cron precision | Confirmed by Hermes as to minimum precision and whether it can carry scheduling reconciliation |

### B. Roles and Boundaries

| # | Resolution Item | Conclusion |
|---|---|---|
| B1 | Doc/Design Reviewer boundary | All pure-documentation work belongs to it (design + work-package + document review); **low-risk design needs no review, high-risk design reviewed by another agent of the same role** |
| B2 | L0 trust | **Trust the developer's L0 evidence** (subsequent integration / full test serves as fallback); Reviewer does not re-run the full unit-test suite |
| B3 | Trigger integration | **Keep V2.4 behavior**: TaskAccepted → create an independent integration task |
| B4 | Carrier–role mapping | **No fixed mapping**; carrier agent and model are specified at the **design stage** in the detailed task list (prices fluctuate, assigned ad hoc) |

### B5. Role Consolidation (new resolution in this review, implemented in README §4 / §6.1 / prompt templates)

| Consolidated Role | Core Responsibility | Key Boundary |
|---|---|---|
| Implementer (development) | First-time development + rework + L0 unit tests | Only in own feature branch; runs L0 and records evidence; does not approve own work |
| Reviewer (review) | Review development output (diff/commit) | Based on diff + L0 evidence, only spot-checks critical paths, does not re-run the full suite; V3.0 Code Immutability Constraint (danger-full-access + isolated verification workspace), does not merge |
| Integrator (integration) | Merge feature → iteration | Only merges precisely approved commits; resolves conflicts; does not review own merge |
| Validator (test verification) | Full test after merge (L1 + regression) | Runs full suite on integration branch; read-only / test environment; does not merge |
| Doc/Design Reviewer (documentation/design review) | All document review + design + work-package/context | Pure documentation belongs to it; low-risk design not reviewed, high-risk reviewed by another of same role |
| Coordinator (total scheduling) | Dispatch / periodic reconciliation / gate judgment | Carried out by Hermes; does not perform sub-agent hands-on work |

Consolidation sources: Reworker → Implementer; System Reviewer → Reviewer; Main Merge → Integrator; Requirements/Design/Task three reviews → merged into one Doc/Design Reviewer.
Hard floor retained: development separated from review; merge execution separated from review; final approval never self-attested by the developer.

### C. Process Orchestration

| # | Resolution Item | Conclusion |
|---|---|---|
| C1 | Dispatch protocol-header design | Designed by Hermes (principles: save tokens, introduce no risk, clear semantics) |
| C2 | Parallelism | **Allow multiple tasks in parallel**; tasks with conflicts are not set to parallel; if a genuine conflict arises during execution → escalate to Richy for coordination |
| C3 | Evidence persistence format | Designed by Hermes (same principles as C1) |
| C4 | Rework review | **Keep behavior: review only the changed parts** (saves tokens) |
| C5 | worktree creation | **Plan Y + Hermes unified orchestration**: Implementer creates it itself (verified feasible under danger-full-access); Hermes supplies TaskID / base branch / worktree directory at dispatch; **does not personally run `git worktree add`** |
| C6 | worktree cleanup | **Unified cleanup by the Integrator that performs the final merge** (D2) |

### D. Git / Sandbox / Security

| # | Resolution Item | Conclusion |
|---|---|---|
| D1 | danger-full-access usage | Available to any role that needs commit permission (development / integration / design) |
| D2 | worktree lifecycle | Creation = Implementer (Plan Y); use = Implementer; ~~review = Reviewer read-only~~ → V3.0: Reviewer uses an isolated detached verification workspace; cleanup = Integrator unified |
| D3 | How Reviewer obtains snapshot | ~~**D3b: Reviewer enters the developer's worktree directly for read-only review**~~ (superseded by V3.0: changed to isolated detached verification workspace + Code Immutability Constraint, see 09 §7.3) (strictly serial, non-overlapping; developer read/write, reviewer read-only; one development agent owns one feature; avoids losing git-excluded local content across worktrees) |

### E. Responsibility for Requirements / Design / Task Breakdown

| # | Resolution Item | Conclusion |
|---|---|---|
| E1 | Requirements analysis / boundary partitioning / detailed design / task breakdown | **All completed by Richy + GPT-5.6-Sol**; Hermes does not lead |
| E2 | Hermes's role on development tasks | **Review** whether reasonable / executable, feed back to Richy, Richy + GPT revise. Hermes does not decide core design decisions |

### F. Ledger and Notification

| # | Resolution Item | Conclusion |
|---|---|---|
| F1 | Ledger presentation | **No new development dashboard**; Hermes maintains the ledger as the source of truth (SQLite / file) |
| F2 | Ledger notification | **On ledger update, push to TG with key nodes + 0 token + ledger link**: read ledger structured fields + template assembly (no LLM), push only on key states (Submitted/Approved/ChangesRequested/Integrated/needs-authorization/abnormal-failure); details viewed via link on demand |
| F3 | Iteration advance / Release | **Requires Richy's authorization**; passing Level 1 + regression does not mean it can go live; real-environment version acceptance is performed personally by Richy in the real test environment (consistent with V2.4 Level 2/3 being run + authorized by the project owner) |

---

## III. Context Package (Context L2) Responsibility and Staged Generation (new resolution in this review)

- **Owner**: Doc/Design Reviewer.
- **Generation timing**: before development (at dispatch time), and **dependencies already satisfied**.
- **Rules**:
  1. After upstream baselines are ready and before development dispatch, generate the Context L2 skeleton (goal / risks / established facts / exploration scope / modification boundary / prohibited items).
  2. Only after a dependency task has completed (dependencies satisfied at that moment) may its verified output / evidence be incorporated into the package; **never pre-write content a dependency has not yet produced**, to avoid contradicting actual code.
  3. Incremental supplementation after dependency completion remains the Doc/Design Reviewer's responsibility (reusing the evidence-persistence mechanism).
- **Core principle**: Context L2 is generated before development (dependencies satisfied); prefer leaving "to be supplemented after downstream output" over writing placeholders that may conflict.

> Note: This resolution also closes a gap in V2.4 spec 06, which did not specify "who generates the work package" (the template only had a `GeneratedBy:` field name). Filled in here.

---

## IV. Layered Test Responsibility (prevent token waste, new resolution in this review)

| Layer | Responsible Role | Action |
|---|---|---|
| Level 0 (unit/component) | Implementer | Run once after development, results recorded in evidence directory |
| Review | Reviewer | Does not re-run full unit tests; reviews based on diff + L0 evidence, only smoke-tests critical paths |
| Level 1 + regression | Validator | Run the full suite only once before merge |

Hermes injects test boundaries at dispatch (explicit in prompt):
- To Implementer: "After implementation, run Level 0 unit tests; submit the test results as evidence for the Reviewer to inspect."
- To Reviewer: "The developer has run Level 0 unit tests (see evidence). You only need to review the diff logic + smoke-test critical paths. Do not re-run the full unit-test suite."

Core principle: tests are run once by the responsible party, others review based on evidence; integration / regression is run only before merge (adopts deepseek outline G-3 evidence persistence as the reuse carrier).

---

> Revision: 2026-08-20, Hermes — cleaned up dangling book-title brackets referencing archived historical documents.

---

*This document is a record of the Hermes migration discussion; concrete execution is implemented by Hermes per the subsequent "Hermes Capability Boundary List" and the dispatch mechanism.*


---

## V3.0 Appendix (revised by Tiffany-Dev, 2026-08-24): Scheduling-Concurrency Control Model

V2.5 assumed "single-machine single instance, no coordination-lease lock"; from V3.0 onward, scheduling authority may be competed for by the coordinating cron and interactive sessions.
The concurrency model is now **CoordinatorEpoch (FencingToken)**:

- At any moment there is exactly one Epoch owner; dispatch/consumption from an old Epoch is always rejected;
- Takeover = atomic conditional update (effective only when Epoch+StateRevision match);
- Normal handover requires dual acknowledgement; after the old instance is lost beyond LostOwnerTimeout, recovery takeover requires Richy's authorization;
- Full protocol in 09 §12.3; carrier-policy change channel in 09 §12.4 and 05 §6.3.
