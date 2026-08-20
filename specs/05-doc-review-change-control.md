# Document Review & Change-Control Spec

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Applies to: background analysis, requirements specification, detailed design, detailed development task, and their subsequent revisions
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Finalized: 2026-08-11
> Revised: 2026-08-20
> Reviewer: Richy (approved)

## 1. Purpose

This spec ensures the project improves gradually through multiple rounds of human–machine discussion, without losing its formal baseline due to chat context, verbal confirmation, or downstream implementation changes.

The goal of document review is not to finish writing once, but to let every important decision go through:

```text
Propose → Record → Analyze impact → Pending review → User confirms → Write into baseline → Propagate downstream
```

## 2. Review Authority and Responsibility

### 2.1 Project Owner

- Holds approval authority over business scope, user behavior, risk acceptance, architecture conflicts, and the final document;
- May approve, reject, or request further comparison of approaches;
- Only the project owner can declare a document approved overall;
- May authorize specific independent execution tasks to perform code, database, external-system, and Git operations, but the authorizations do not automatically include each other; database or persistent-state authorization must not be transferred to CodingTask, ReworkTask, Reviewer, or ordinary merge tasks.

### 2.2 Execution Agent (WorkBuddy / Codex)

- Responsible for investigation, raising questions, comparing approaches, editing documents, and maintaining status;
- For obvious errors, provide evidence and reasoning; do not mechanically please;
- Must not approve important trade-offs on the project owner's behalf;
- Must not automatically treat the user's exploratory expressions as a final decision;
- After user confirmation, must write the conclusion into the formal document, not merely reply "understood" in chat.

### 2.3 Development and Review Personnel

- Use the latest "Approved" version as the baseline;
- When implementation conflicts with the baseline, stop the relevant part and submit a change request;
- Must not silently modify requirements or design via code facts in reverse.

## 3. Review Methods

### 3.1 Item-by-Item Review

Each document should provide a pending-review list at the end. Recommended format:

```markdown
- [ ] DEC-001: one business record maps to one search document.
- [ ] DEC-002: lists use derived summary, detail uses database as source of truth.
```

After user confirmation, update to:

```markdown
- [x] DEC-001: passed, confirmed 2026-08-11.
```

On confirmation, the body must also be updated; do not only check the list while the body still retains the old approach.

### 3.2 Section-by-Section Review

Complex documents may proceed as follows:

1. The execution agent first submits a complete draft or one logically-complete section;
2. List the decisions needed this round;
3. The user gives opinions item by item;
4. The execution agent modifies the body, decision records, and pending-review list;
5. The execution agent states the remaining unreviewed sections;
6. After all sections are complete, request overall review.

### 3.3 Overall Review

Before marking "Approved," the following must be confirmed:

- Pending-review checkboxes are zero;
- Unresolved issues are not hidden in the body;
- Upstream baseline version is correct;
- Body, examples, diagrams, acceptance checklist, and decision records are consistent;
- Scope and non-scope are explicit;
- Document version and update time are updated;
- The project owner has explicitly stated overall approval.

## 4. Decision Records

The following decisions must be kept on record:

- Changing data granularity, identity, or source of truth;
- Changing user-visible behavior or business caliber;
- Changing scope, compatibility, or consistency;
- Selecting or rejecting an important architecture approach;
- Accepting an obvious performance, cost, security, or operations trade-off;
- SOLID conflicting with existing architecture;
- Cancelling or deferring a capability originally in this cycle.

Standard format:

```markdown
### DEC-001: <decision title>

- Status: Pending review / Passed / Superseded
- Background:
- Options:
- Decision:
- Rationale:
- Impact scope:
- Confirmer:
- Confirmation time:
- Supersession: none / DEC-000
```

Deleting an old decision loses design history, so when overturned it should be marked "Superseded" and linked to the new decision.

## 5. Versioning Rules

`Vmajor.minor` is recommended:

- Small draft supplement: bump minor, e.g. `V0.2 → V0.3`;
- First overall approval: enter `V1.0`;
- Post-approval new but compatible rule or design: `V1.1`;
- Changing core scope, business semantics, data identity, or incompatible design: `V2.0`;
- Pure typo and semantics-preserving formatting fixes need not bump version, but should update the time.

The current version must be recorded at the top of the document; upstream document references must carry a version, not just a file name.

## 6. Upstream Change Propagation

### 6.1 Impact Relations

```text
Background change
  → may affect requirements, design, task

Requirement change
  → must check design, task, and existing code

Design change
  → must check task, context package, prompts, tests, and existing code

Task-orchestration change
  → usually does not reverse-change requirements and design, but may expose design gaps

Verification or real-run defect found
  → must judge whether implementation, test, design, requirement, context, or spec has a gap
```

### 6.2 Handling Steps

1. Record the change request and new decision in the upstream document.
2. Judge whether that document needs re-review.
3. Mark all affected downstream documents as "Pending Sync."
4. Build an impact list: sections, tasks, code, scripts, tests, and manual steps.
5. Update downstream body, traceability matrix, and version.
6. Re-review item by item for material changes.
7. Downstream may continue related development only after re-passing.
8. Invalidate affected context packages, Review conclusions, test evidence, and merge approvals until re-verified.

Do not bypass requirements or design changes by only modifying the detailed development task.

## 7. Change Requests During Development

When the following occur during development, pause the relevant task:

- Requirement rule cannot be determined;
- Detailed design clearly conflicts with existing code or platform behavior;
- Need to modify the prohibited scope;
- Performance estimate differs from measurement enough to change the approach;
- External platform version or limit does not match the design;
- Must change user-visible behavior to pass tests;
- Security or data-loss risk found.
- Context package inconsistent with current code, production entry, or dependency graph;
- Real-run reveals system behavior not covered by automated tests;
- Unauthorized code change, or a change that alters requirements, design, scope, or risk acceptance; an authorized Review rework is not this item and follows the original task-branch rework loop.

Change-request format:

```markdown
### CR-001: <title>

- Discovery stage:
- Discoverer:
- Current baseline:
- Facts and evidence:
- Affected requirement/design/task:
- Options:
- Recommendation and rationale:
- Current task status: Blocked
- Project owner decision: pending review
```

Details that affect only internal implementation and do not change approved behavior, boundary, or risk may be handled by the scheduler/reviewer within design constraints, and recorded in the task evidence.

## 8. Status Consistency

### 8.1 Document Status

The document's first-page status, end review checklist, and README index must be consistent.

### 8.2 Task Status

The detailed-task section, the master state table, and the structured task-state-exchange area within the same document must be consistent. There must be no case where the task body says `TaskAccepted` while the master table still says `InProgress`.

The project must specify both a unique `CanonicalTaskDocumentPath` and the Hermes ledger `LedgerLocation`. The former holds the Git-managed task definition and most recent persistent snapshot; the latter is the Hermes single-instance-maintained, Git-excluded sole real-time source of truth for task runtime state (local JSON state file + optional SQLite). Chat context, UI state, cross-task read API, document snapshot, and derived ledger must not override a higher `StateRevision` in the ledger.

Routine updates to the state-exchange area do not change the immutable task definition identified by `TaskDocumentBaselineRef`, nor automatically invalidate code candidate, test, or Review. Task-definition changes outside the state-exchange area must still be reviewed per this spec.

When the ledger is missing, corrupted, version-rolled-back, or cannot explain the task-vs-Git difference, Hermes follows the recovery protocol in `09-hermes-ledger-runtime.md`. Recovery prefers the last valid snapshot and Git in the development-task document; only when the recovered record is explicitly marked `PendingConsumption` or evidence is missing does it read the corresponding `ExecutionRef` directionally—do not recover as a full cross-carrier scan.

### 8.3 Code Status

- Document passed does not mean code is complete;
- Code committed does not mean task is accepted;
- All tasks accepted does not mean real environment is verified;
- Real environment verified does not mean release is authorized;
- Pushed feature branch does not mean merged into the project default main branch.
- `TaskAccepted` does not mean merged into the iteration development branch;
- `Integrated` does not mean passed post-merge integration verification;
- `IntegrationVerified` does not mean the iteration candidate is `ComponentVerified`;
- `TaskAccepted` does not mean `ComponentVerified`;
- `ComponentVerified` does not mean `SystemVerified`;
- `SystemVerified` does not mean `ExternalVerified`;
- A passed Review is valid only for the precise commit recorded.

These states must be stated separately.

## 9. Evidence Requirements

### 9.1 Document Evidence

- Code path and necessary line numbers;
- Data query time, environment, and caliber;
- Log file and key run identifier;
- Official-source link and verification date;
- User-explicitly-confirmed decisions.

### 9.2 Development Evidence

- Actually modified files;
- Build and test commands and results;
- Not-run tests and reasons;
- Static dependency or architecture check;
- Key failure-scenario tests;
- Reviewer conclusion.
- Baseline commit, candidate commit, and review commit;
- Test stubs and what they cannot prove;
- `NotRun` blocking stage, owner, and exception-approval record.

The minimal identity fields for a real coding task are `TaskBranch`, `CodeBaseSHA`, `HeadSHA`, and `CodeBaseSHA..HeadSHA`. Worktree path, file-modification time, chat summary, and UI state may only serve as navigation or auxiliary evidence, not substitute for commit/tree/diff content identity. The old name `BaseSHA` is deprecated: `BaseSHA` appearing in V2.3 artifacts is treated as the compatible old name of `CodeBaseSHA`, for historical read-compatibility only; from V2.5 new artifacts must use `CodeBaseSHA` and must not output `BaseSHA` again.

When the approved requirements, design, or task document is not in the code `CodeBaseSHA`, immutable `RequirementsBaselineRef`, `DesignBaselineRef`, and `TaskDocumentBaselineRef` must be registered separately. Do not use a continuously-changing main-workspace path to substitute for document-content identity.

Do not falsely report real database, search-engine, or other external-environment tests. When no environment exists, write "not executed, pending manual verification by project owner."

### 9.3 Evidence Validity Scope

- Tests and Review must bind repository, unambiguous review target, environment, and time;
- The same fast test on the same review target may reuse results, but the Reviewer should still spot-check or re-run by risk;
- After the review target changes, affected evidence automatically invalidates; whether a pure-document change invalidates code tests is judged and recorded by the Reviewer;
- Developer tests prove reproducibility, Reviewer tests prove independence; the two should not replace counter-example design through mechanical repetition;
- Logs, screenshots, and Agent summaries must trace to the original command or run record.
- The hash algorithm in evidence must be named accurately; SHA-1, SHA-256, Git commit ID, and Git tree ID must not be conflated;
- Evidence documents must not make their own hash a precondition for their own validity, nor form a self-referential loop of "updating evidence invalidates the candidate";
- Verification commands must be directly copy-executable, or reference version-controlled and frozen-version scripts; commands broken by line-wrap are not reproducible evidence;
- File-modification time can only assist in judging sequence, not prove file content is identical;
- Pure-document, ledger, or evidence changes with unchanged code `HeadSHA` do not automatically invalidate existing code tests and Coding Review; the Reviewer only needs to judge whether that document change altered the review baseline.
- Real coding tasks must register `ReviewedCommitRange`, `ReviewedCommitSet`, and `IntegrationMethod`; the actually-integrated commit set must match the reviewed set.
- Evidence recovered from a lost ledger must be marked `Recovered`, with source, original time, recovery time, fingerprint, and confidence; it must not be disguised as an originally-received real-time event.
- Different sources prove only their own responsibility scope: Git proves commits and containment; Reviewer final proves Verdict; the task system proves execution history; user message or formal record proves authorization. No source may overstep to infer other facts.
- Complete subtask results provided by the user should be registered and verified first; when the original task cannot be read, enter `PendingVerification`, not overridden by a UI label, and not advanced through a gate without verification.

## 10. Merge-Candidate Freeze and Approval Validity

### 10.1 Candidate Freeze

Before entering Coding Review, Level 1 overall Review, Level 3 final acceptance, or final merge review, an unambiguous candidate target must be recorded. Real coding tasks default to Git commit and `CodeBaseSHA..HeadSHA`; iteration and version candidates default to commit/tree. Pure-document, investigation, or pre-approved special tasks may use PR revision, diff hash, or stable workspace snapshot. Freeze is not a prohibition on fixing, but a rule: a material change to the target produces a new candidate and invalidates affected conclusions.

Active-task ledger, temporary reconciliation time, Agent state, and the evidence document itself must not enter the code-candidate hash. Candidate evidence may reference the candidate SHA, but must not change the referenced candidate's identity by modifying itself.

The development-task document state-exchange area holds real-time state; long logs and backups may reside in the project-configured non-candidate directory. If a state-validation tool is provided, document lock, version review, state transition, and event deduplication must be done through the tool. Specific fields, recovery, and Canary follow `09`.

### 10.2 Branch Roles

The project must configure an actual branch strategy and retain equivalent state and evidence gates. `08` provides a recommended default mapping, but does not restrict trunk-based, release-branch, or feature-flag other strategies.

### 10.3 Final Approval

The final merge reviewer may only give `MergeApproved` or `MergeBlocked` on the frozen candidate, and must not perform the merge. The project owner's merge authorization must be later than the valid review conclusion; the merge target, source, and candidate must match precisely. Only an independent `MainMergeTask` may perform the main-branch merge, and must not smuggle in development changes. After merging, a separate decision on release is still needed.

The iteration branch also observes responsibility separation: after the Coding Reviewer gives `TaskAccepted`, only an independent `IterationMergeTask` may merge per `ReviewedCommitSet` and `IntegrationMethod`. If a merge conflict needs an implementation change, it must be returned to the original task or a fix task created and re-Reviewed; an approved squash must provide equivalent-diff evidence.

## 11. Real-Defect Feedback Loop

The following must trigger a document-and-process retro: approved requirement/design conflicts with reality, same-type defects recur, tests or Review show systematic misses, environment differences cause high-impact problems, or security/data/compatibility/incident involvement. Ordinary local Bugs may be noted in task evidence without expanding the retro.

When a retro is needed, judge:

| Problem type | Must retro |
|---|---|
| Requirement gap | background, requirement, acceptance level |
| Design gap | lifetime, resource, exception, test strategy |
| Task gap | split, junction responsibility, authorization scope |
| Context gap | known facts, open items, autonomous-exploration scope |
| Review miss | role prompt, counter-examples, evidence gate |
| Environment difference | Level 1–3 definition and NotRun decision |
| Generic process defect | this directory's specs and prompt templates |
| Scheduling state loss, duplicate dispatch, or event miss | `09`, Coordinator/reconciliation template, and scheduling Canary |

Generic specs absorb only rules reusable across tasks; they do not hard-code single-incident details.

## 12. Conflict Handling

When the following conflict, the execution agent must not choose on its own:

- User's new opinion vs. approved decision;
- SOLID constraint vs. existing code structure;
- Performance goal vs. platform free quota;
- Data correctness vs. simplified implementation;
- This-cycle scope vs. extra refactoring needed for implementation;
- Automated-test requirement vs. missing real environment.

The execution agent should provide: facts, conflict point, at least two viable options, respective impact, and recommended rationale, and let the project owner decide.

## 13. Document Archiving

- When a new version completely supersedes the old document, the old is marked "Superseded," keeping the link and reason.
- Temporary investigation records may be archived, but their confirmed conclusions must enter the formal document.
- Historical decisions that have affected implementation should not be deleted.
- The README should point to the current effective version and recommended reading order.

## 14. Overall Review Gate

- [ ] Each artifact has passed its local review checklist; not re-checked item by item here;
- [ ] All pending-review items are zero, upstream and downstream versions consistent;
- [ ] Context L2, Review, and test evidence match the current review target;
- [ ] Approved, coded, accepted, manually-tested, merged, and released are not conflated;
- [ ] Frozen candidate is consistent with Review, test, and merge approval;
- [ ] Code baseline and applicable requirements, design, task-document baselines are all unambiguous;
- [ ] The development-task document state-exchange area is unique, parseable, and version-monotonic; derived audit cache does not enter candidate hash;
- [ ] Recovered evidence, user-imported evidence, and original-event sources and confidence are not mixed;
- [ ] Real defects that hit the trigger condition have completed document-and-process retro.

## 15. Definition of Done

Review is truly complete only when the formal document has saved all key decisions, states, and evidence, upstream and downstream baselines are consistent, and the project owner has explicitly approved. Verbal consensus in chat that is not written back to the document is not a reusable project spec.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.4 | 2026-08-15 | — | Introduced shared JSON state source of truth and RecoveryOnly |
| V2.5 | 2026-08-20 | WorkBuddy | §2.2 Codex→Execution Agent; §8.2 shared JSON→Hermes ledger; §9.2 BaseSHA marked deprecated |
| V2.5 | 2026-08-20 | Hermes | Review revision: §3.2/§12 unified standalone Codex into Execution Agent |
| V2.5 | 2026-08-20 | Hermes | Cleaned V2.4 remnants: removed "lease" with no corresponding concept (single-instance, no coordination-lease lock) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
