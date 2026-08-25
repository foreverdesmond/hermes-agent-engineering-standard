# Verification, Code Review & Merge-Gate Spec

> Spec version: V3.0
> Document status: Approved (V2.5 is the previous baseline; finalized 2026-08-24, Richy final review passed)
> Applicability: all development tasks needing code Review, integration verification, branch merge, or real-environment acceptance
> Author: Tiffany-Dev (V3.0 revision; initial version = WorkBuddy)
> Revised: 2026-08-24
> Reviewer: Richy (approved)

## 1. Purpose

This spec distinguishes local correctness, component integration, production-form system runtime, and the real external environment, and defines for each layer its executor, evidence, branch action, and rework loop.

## 2. State Gates and Default Branch Strategy

Level 0–3 are generic verification states, not dependent on any one Git branch model. The following is the recommended default mapping:

| Logical role | Role |
|---|---|
| Task branch | Single-task implementation, rework, and development commit |
| Iteration development branch | Independent integration task aggregates precisely-approved Level 0 commits |
| Long-term integration branch | Real system verification after Level 1 passes |
| Stable branch | Stable candidate after Level 3 and final review pass |

Projects may use trunk-based, short-lived branches, release branches, feature flags, or other strategies, and may merge some physical branches, but must retain equivalent candidate isolation, evidence, rollback, and authorization gates. Real coding tasks default to an independent task branch; deviating from the default strategy requires prior project-owner approval. The actual branch strategy is configured by the development task.

### 2.1 Git Permission Boundary

- Development and rework may only create commits on their own task branch;
- Coding Reviewer, System Reviewer, and final merge Reviewer are subject to the **Code Immutability Constraint**: unified use of danger-full-access to obtain build and test capability, but they must not modify tracked business source, must not commit candidates, and must not merge; execution must be in an isolated detached verification workspace based on the precise candidate commit; verify the candidate's HEAD/tree two-way before and after review—any change invalidates that Review conclusion (see 09 §7.3);
- Only an independent iteration-integration task may merge the precise `TaskAccepted` commit into the iteration development branch;
- Only an independent main-branch merge task may merge the precise candidate into the main branch after `MergeApproved` and explicit authorization;
- A Reviewer must not also be the merge executor for the same candidate; any action entering the shared iteration, long-term integration, or main branch is executed by an independent merge task.

## 3. Level 0: Single-Task Verification

### 3.1 Goal

Prove that one task is locally correct, buildable, testable, and design-conformant within the approved scope.

### 3.2 Execution

- The developer runs directed compilation, unit tests, and affected component tests;
- The developer commits on the unique task branch and records `CodeBaseSHA`, `HeadSHA`, and `CodeBaseSHA..HeadSHA`;
- An independent Coding Reviewer checks the actual review target, re-runs the minimal necessary tests, and forms an independent risk hypothesis; when an important executable risk exists, verifies a counter-example, otherwise records the basis for no extra counter-example;
- After rework, the original Review conclusion is invalidated and re-reviewed.

#### 3.2.1 Review Execution Status vs. Verdict

Review must separate execution status from code conclusion:

```text
ExecutionStatus: Completed / Blocked
Verdict: Approved / ChangesRequested / NotIssued
BlockerType: None / RepositoryEnvironment / ToolRuntime / Authorization
```

- `ExecutionStatus: Completed` is required before `Approved` or `ChangesRequested`;
- When `ExecutionStatus: Blocked`, `Verdict` must be `NotIssued`;
- Test `NotRun` does not automatically equal Review block. The Reviewer may give `ChangesRequested` based on verifiable code paths, design deviation, or an independent counter-example even when tests cannot be re-run;
- Tool-call format, task reading, execution-protocol, or model-runtime anomalies belong to `ToolRuntime`, and must not be registered as code Findings or repository-environment defects;
- Only unparseable review commits, Git objects, or necessary repository files belong to `RepositoryEnvironment`;
- Needing unauthorized real external operations belongs to `Authorization`;
- Any Verdict must bind `InvocationID`, the Reviewer's execution identity, and the precise review target.

Control-plane execution failure recovery and duplicate-instance handling follow `09-hermes-ledger-runtime.md`, and are not counted in the rework rounds of the same code Finding.

### 3.3 Pass Condition

- Requirements, design, SOLID, scope, error, and security gates pass;
- All tests that block task acceptance pass;
- The development commit and the Reviewer's review target the same unambiguous commit;
- State updated to `TaskAccepted`.

### 3.4 Branch Action

`TaskAccepted` only grants eligibility to enter an independent iteration-integration task; it does not automatically mean the code is merged. The integration executor must merge the precise `HeadSHA` actually reviewed by the Reviewer; after success, record `Integrated`. After merging, the **Integrator (or an independent `IntegrationValidationTask`) triggers and reports affected integration-check results** (through its carrier channel, reporting a structured protocol header containing `IntegrationCommit` + `IntegrationStatus`, consumed by Hermes into the ledger); `IntegrationVerified` is recorded only after the affected checks pass. This must not be claimed as the version having passed Level 1–3.

## 4. Level 1: Iteration Component Integration

### 4.1 Goal

Prove that after all planned tasks are combined, automated component behavior and cross-task wiring conform to the approved baseline.

### 4.2 Preconditions

- Each task, after entering the candidate, has run affected continuous-integration checks;
- All planned tasks included in the candidate are `Integrated`, and affected checks reach `IntegrationVerified`;
- The iteration candidate commit is frozen (**candidate-freeze event**: the project owner freezes and informs Hermes, or Hermes polls the freeze marker; this human-gated event triggers Hermes to auto-dispatch the Level 1 Validator / System Reviewer, see `09` §6.1);
- No unexplained workspace changes;
- The system-integration responsibility task is complete.

### 4.3 Coverage

- Solution build and full automation;
- Component, contract, Fake/Mock/in-memory integration;
- Safely-automatable registration and key-entry activation in the production composition root;
- Cross-task interface, config, state, and compatibility;
- System-integration Reviewer's independent risk hypothesis and counter-examples as needed;
- Test-stub proof boundary and Level 2/3 `NotRun` list.

### 4.4 Pass Condition and Branch Action

After the system-integration Reviewer gives `ComponentVerified` and the project-configured `NotRun` blocking Level 1 is zero, execute the project branch strategy; by default, entry into the long-term integration branch is allowed.

Level 1 is not proof that the production external environment passes. It allows subsequent real testing to find problems and rework.

## 5. Level 2: Production-Form System Verification

### 5.1 Goal

Verify production-form entry, config, Scope, concurrency, persistence call chain, and key system behavior on the project-designated integration candidate.

### 5.2 Execution Mode (V3.0: clean-checkout homomorphic gate)

L2 verification must be executed on an **independent detached worktree** checking out the **precise candidate commit**, and must not reuse any developer worktree (protecting its uncommitted content, and also ensuring the verification environment is homomorphic with the delivery environment):

- After checkout and before and after testing, the candidate's **entire tracked content** (business source, config, project files, test assets) must remain zero-change—the verification process may only produce untracked test artifacts that are cleaned up in a controlled way;
- Ignored host-config-type dependencies must declare a **reproducible controlled fallback source** (such as a sanitized sample template versioned with the repository); without a controlled fallback source, form an explicit "environment/config blocker" record (with a responsible owner) and escalate to the project owner; do not keep blindly retrying;
- Test artifacts may be cleaned up in a controlled way; real external dependencies (production DB/storage, external credentials, real APIs) are out of scope at this level; anything not run is `NotRun`.

#### 5.2.1 Execution Options

The project may choose manual, semi-automatic, or automatic. Default recommendation: the Agent prepares config and steps, the project owner authorizes and executes, the Agent analyzes logs and evidence.

Level 2 may use the existing integration environment and recoverable business data. Whether writes are allowed is approved by the project owner based on the nature of the project data.

### 5.3 Database and Persistent-Resource Safety

- Any verification that may modify a database, index, storage, or real external state must not be done by development, rework, Coding Review, system Review, or merge-execution tasks; an independent verification task must be used, with the project owner explicitly authorizing the execution subject;
- By default, Agents are prohibited from automatically creating, deleting, truncating, or rebuilding databases, schemas, persistent volumes, indexes, or external resources;
- Temporary databases or containers must not be made a mandatory condition for all projects;
- Any destructive operation needs a precise target, impact description, and single explicit authorization;
- When unauthorized, record `NotRun` or use non-destructive manual verification;
- Do not use unresolved variables, wildcards, or broad directories to perform deletion;
- Recoverable test data does not automatically grant operation authorization.

### 5.4 Minimal Evidence

- Branch, commit, environment, and config summary;
- Execution steps and run identifier;
- Success, failure, cancellation, and concurrency observations;
- Logs, output, or data phenomena;
- Found problems, not-run items, and risk conclusions.

### 5.5 Failure Loop

After Level 2 finds a problem:

```text
Record defect and reproduction evidence
→ Develop on task/fix branch
→ Level 0 independent Review
→ Run Level 1 regression per impact
→ Merge back into long-term integration branch
→ Re-run affected Level 2 scenarios
```

On pass, state is `SystemVerified`. Whether to trigger a branch action is decided by the project strategy; by default, stabilize the candidate on the long-term integration branch.

## 6. Level 3: Real External-Environment Version Acceptance

### 6.1 Goal

Verify real external systems, network, permissions, scale, long-running behavior, and user-visible results that local automation cannot adequately substitute.

### 6.2 Execution

- Usually requires project-owner authorization;
- May be executed manually or semi-automatically;
- An Agent may inspect config, make a plan, analyze logs, and form a report;
- Must not connect to or modify real external systems without authorization.

### 6.3 Pass Condition

- Key Level 3 scenarios specified by requirements have a conclusion;
- Key business results, performance, failure, and recovery conform to approved standards;
- Remaining `NotRun` has approved exceptions and recorded risk;
- State is `ExternalVerified`.

After passing, execute the project-approved stable-candidate strategy; by default, applying for stable-branch merge is allowed. Tasks with no real external dependency may mark Level 3 as `N/A`, but must state the reason. The project may pre-approve common `N/A` categories; only simplifications beyond the pre-authorized boundary need item-by-item approval.

## 7. Final Merge Review

The final merge reviewer must check:

- Source, target branch, and unambiguous frozen candidate;
- Level 0–3 states and applicability;
- Whether Review and tests bind the current commit;
- Whether there is still a `NotRun` blocking the target branch;
- Whether documents, config, migration, and workspace are consistent;
- Whether code changes occurred after freeze without review;
- Whether the project-owner authorization still needed for final execution is listed as an independent gate.

The conclusion can only be `MergeApproved` or `MergeBlocked`. The final merge reviewer only reviews eligibility, and does not execute the merge.

The independent main-branch merge task may execute only when all the following are met:

- `MergeApproved` still binds the current precise candidate;
- The project owner has given explicit authorization for source, target, and candidate;
- Workspace and branch state are consistent with the review;
- The merge does not need to change the reviewed implementation.

When a conflict-resolution change to the implementation is needed, `MergeApproved` is invalidated; a fix candidate must be created and re-verified. The merge-complete state is `Merged`, which does not automatically become `ReleaseReady`.

## 8. Test Duplication and Evidence Reuse

Both the developer and the Reviewer running tests is necessary independent verification, but mechanical repetition should be avoided:

- The developer runs targeted tests at the change point and affected tests;
- The Reviewer re-runs the minimal relevant set, spot-checks key results, and adds counter-examples;
- Level 1 uniformly runs the full automation on the frozen candidate;
- Level 2/3 only repeat the fast tests needed to establish a baseline, focusing on verifying new environment risks;
- Stable results for the same review target, command, and environment may be referenced, but original evidence must be retained;
- After the review target changes, evidence is invalidated by impact;
- Test output should be concise by default, keeping only summary and failure details, to reduce time and context consumption.

The Reviewer's main value is independent reasoning and counter-examples, not copying all the developer's work.

Choose the minimal test scope by change type:

| Change | Default action |
|---|---|
| Same `HeadSHA`, only activity-ledger or evidence-formatting change | No repeated code Review or test; only verify evidence validity |
| Same `HeadSHA`, material change to requirements/design review baseline | Judge whether original Review is invalidated; re-review by impact |
| Task-branch code produces new `HeadSHA` | Run affected Level 0 Review and tests on the new `CodeBaseSHA..HeadSHA` |
| First time an approved task merges into the iteration branch | Run affected continuous-integration checks |
| Iteration candidate freeze or combination changes | Run Level 1 full automation and system Review |
| Only manual/external-environment evidence update | Only re-run affected Level 2/3 scenarios; do not mechanically repeat Level 0/1 |

A known baseline failure must be registered with test, first evidence, impact judgment, and owner. When the candidate does not touch the relevant path, the baseline may be referenced; do not re-investigate every round; when this change may affect that failure, re-verify.

## 9. NotRun, Exceptions, and Waiver Boundaries (V3.0 extension)

### 9.1 NotRun Record Requirements

Each `NotRun` must record:

- Acceptance item and Level;
- Reason;
- Risk possibly missed;
- Which step among task, long-term integration branch, stable branch, or release it blocks;
- Owner and planned time;
- Alternative evidence;
- Exception approver and time.

Lack of environment does not automatically lower the acceptance level. The project owner may accept risk, but it must not be rewritten as `Passed`.

### 9.2 Waiver Boundary Declaration (V3.0 new)

The dispatch of a verification-class task must include a **waiver boundary declaration**, explicitly listing the failure modes at this verification level that are "known to exist but ruled by the project owner not to block this level's conclusion." Each waiver must satisfy the complete seven-item tuple; missing any item makes the waiver invalid:

| # | Field | Description |
|---|---|---|
| 1 | Command/scenario | The precise command or verification scenario that triggers the failure |
| 2 | Failure signature or missing dependency | Precise error signature (including error code + context); using only a generic error code is prohibited |
| 3 | Impact scope | The components/paths affected by the failure, and the parts explicitly unaffected |
| 4 | Risk | Issues that may be missed after waiving |
| 5 | Alternative evidence | What other evidence compensates for this unverified item |
| 6 | Authorizer | Project owner (Richy) + date |
| 7 | Expiry candidate identity | The candidate SHA this waiver is bound to; if the candidate changes, it expires and must be re-adjudicated |

Rules:

- A generic error code must not alone serve as a waiver signature (V5 lesson: MSB3030 appeared both for missing host config and for macOS signing issues—the same error code with different root causes);
- **Unified status model (V4 correction, Codex pointed out the status-enumeration conflict)**: a pass with valid waivers is recorded as `VerifiedWithWaivers` (new state, added to all state tables and template Status enumerations); `Verified` means full passage without any waiver. Both are passing conclusions, but `VerifiedWithWaivers` does not satisfy gates requiring "no waivers" (such as checklist closure before FINAL-REVIEW);
- The existing "must not be rewritten as `Passed`" rule remains: `Passed` is used only in no-waiver scenarios (or as a colloquial equivalent of Verified);
- Waivers are adjudicated item by item by the project owner and recorded in the runtime ledger; the verification executor must not expand the waiver scope on its own.

## 10. Risk Adaptation

- Low-risk pure-logic tasks may simplify Level 1, and Level 2/3 may be `N/A`;
- When production entry, concurrency, persistence, security, or cross-system are involved, Level 2 must be evaluated;
- When real third-party behavior, permission, network, or scale is involved, Level 3 must be evaluated;
- The project may pre-approve low-risk tracks, common `N/A`, and role combination; only simplifications beyond the pre-authorized boundary need item-by-item escalation.

### 10.1 Unambiguous Review Target

Review and verification may bind:

- Git commit;
- PR revision;
- diff/patch hash;
- Explicitly-recorded stable worktree snapshot.

Real coding tasks must by default bind task branch, `CodeBaseSHA`, `HeadSHA`, `ReviewedCommitRange`, `ReviewedCommitSet`, and `IntegrationMethod`. PR revision, diff/patch hash, or stable worktree snapshot apply only to pure-document, investigation, or pre-approved exceptions. A worktree path cannot substitute for commit identity.

After a material change to the target, the Reviewer must re-evaluate which conclusions and test evidence are invalidated. A new `HeadSHA` automatically invalidates the old Coding Review; only when the development-task document's state-exchange area, reconciliation time, or evidence summary change and the code SHA is unchanged does it not automatically trigger code re-review.

## 11. Evidence Record Template

```markdown
# <Candidate Version> Verification Record

> Level:
> Branch:
> Commit:
> Environment:
> Executor:
> Reviewer:
> StartedAt:
> CompletedAt:
> Verdict:

## 1. Goal and Related Acceptance Items
## 2. Real Path and Stubs
## 3. Command or Manual Steps
## 4. Results and Raw Evidence
## 5. Findings
## 6. NotRun, Risk, and Exceptions
## 7. Branch-Action Suggestion
```

## 12. Review Checklist

- [ ] The project branch strategy is configured, retaining equivalent state and evidence gates;
- [ ] Development, Review, iteration integration, system Review, final review, and main-branch merge permissions are separated;
- [ ] Real coding tasks use an independent task branch and precise `CodeBaseSHA..HeadSHA`;
- [ ] The applicability, executor, and blocking stage of Level 0–3 are explicit;
- [ ] Branch actions configured by the project are executed only after Level 1 passes;
- [ ] Level 2 allows finding problems manually on the project-designated integration candidate and reworking;
- [ ] Level 3 and final merge review satisfy the project stable-candidate gate;
- [ ] Destructive resource operations are prohibited by default, with precise exception authorization;
- [ ] Test duplication is controlled by risk; the Reviewer forms an independent risk hypothesis and verifies counter-examples as needed;
- [ ] Review's `ExecutionStatus`, `Verdict`, and `BlockerType` are separated; ToolRuntime is not treated as a code Finding;
- [ ] Test `NotRun` is not mechanically interpreted as Review block;
- [ ] All conclusions bind an unambiguous review target;
- [ ] `TaskAccepted`, `Integrated`, `IntegrationVerified`, and `ComponentVerified` are not mixed;
- [ ] Test scope is chosen by actual change; known baseline failures and evidence reuse are recorded;
- [ ] NotRun is not disguised as passed.

## 13. Definition of Done

This spec may be used for the iteration's merge judgment only when the project has completed branch-strategy configuration, mapped each acceptance item to Level 0–3, made execution and Review responsibility explicit, obtained approval for the destructive-resource policy, and made NotRun and evidence rules executable.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.4 | 2026-08-15 | — | Introduced Review execution-status vs. Verdict separation |
| V2.5 | 2026-08-20 | WorkBuddy | §3.2.1 reference "09" changed to "09-hermes-ledger-runtime.md" |
| V2.5 (pending review) | 2026-08-20 | Hermes | §3.4 added IntegrationVerified triggered by Integrator reporting integration-check results; §4.2 added candidate-freeze human-gated event (Richy freezes → Hermes auto-dispatches Level 1, consistent with 09 §6.1) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
| V2.5 errata | 2026-08-21 | WorkBuddy | Synced source errata bd6a71f: heading-level, wording, and reconciliation-terminology fixes |
| V2.5 errata 2 | 2026-08-22 | Hermes | §3.4 adds that `Integrated` requires merging the precise HeadSHA into the **current iteration branch with push succeeding**; a local-only/integration branch must not count as Integrated; downstream CodeBaseSHA must be obtainable from the current iteration branch (traceback: SCRAPER-V5-INT-CTX-002 missing unpushed baseline) |
| V3.0-draft | 2026-08-24 | Hermes | V3.0 revision (proposal v5): ① §2.1 Reviewer "read-only" changed to "Code Immutability Constraint"—unified danger-full-access (build/test needs write permission), isolated detached verification workspace + before-and-after HEAD/tree two-way reconciliation, any change invalidates the conclusion; ② new §5.2 clean-checkout homomorphic gate: L2 must verify on an independent detached worktree of the precise candidate commit; gitignore dependencies must declare a reproducible controlled fallback source, missing means environment-blocker escalation; ③ new §9.2 waiver boundary declaration seven-item tuple (command scenario / failure signature / impact scope / risk / alternative evidence / authorizer / expiry candidate identity), generic error codes forbidden as sole waivers; passes with valid waivers recorded VerifiedWithWaivers |
| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds (proposal v1-v5 plus nine body rounds) closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
