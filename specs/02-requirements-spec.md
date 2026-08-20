# Requirements Specification Spec

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Document positioning: transforms the reviewed problem and business goals into acceptable system requirements
> Prerequisite gate: the Background & Current-State Analysis has been approved
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Originally finalized: 2026-08-11
> Revised: 2026-08-20
> Reviewer: Richy (approved)

## 1. Document Goal

The requirements specification defines "what the system must do," and becomes the common baseline for detailed design, development scope, and final acceptance.

The requirements document is not responsible for deciding class names, directories, SDK call style, SQL details, or specific algorithm implementations; only when a technical constraint itself is a business, compatibility, security, or platform requirement should it be written into requirements.

## 2. Requirements-Writing Principles

### 2.1 Start from User Outcomes

- Describe the user path and perceivable result first, then the system capability.
- Performance metrics should be tied to real usage scenarios, not just "high performance."
- Architectural elegance cannot substitute for business usability, data correctness, and operational maintainability.

### 2.2 Acceptable

Each requirement should include where possible:

- A stable ID;
- Trigger condition;
- Input or applicable scope;
- The result that must be produced;
- Boundaries and exceptions;
- An observable acceptance criterion.

Avoid unverifiable wording such as "as much as possible," "appropriate," "fast," or "reasonable." When quantification is genuinely impossible immediately, state the current baseline, the verification method, and who determines the threshold at which stage.

### 2.3 Explicit Source of Truth and Consistency

When derived data, cache, search index, messaging, or multi-store systems are involved, the following must be explicit:

- Which system is the source of truth for the data;
- Which data may be redundant;
- Where lists and details are read respectively;
- The acceptable inconsistency window;
- Which side wins on conflict;
- Whether degradation is allowed on failure and how the caller is made aware.

### 2.4 Explicit Scope Rather Than Implied Scope

- "In-scope this cycle" should list the capabilities to be delivered.
- "Out-of-scope this cycle" should list content easily mistaken as included.
- When the document provides design input for a future interface or module, it must explicitly state that this is not equal to authorization for current-cycle development.
- Common extension items such as old-system cleanup, global refactoring, multilingual support, API rework, and deployment/go-live must each be explicitly judged as in-scope or not.

### 2.5 Acceptance Levels Must Be Decidable

- Each key requirement must state which of Level 0, 1, 2, 3 proves it; level definitions reference `08-verification-review-merge-gate.md`.
- Requirements must distinguish automation-provable parts, production-form system verification, and real external-environment verification.
- Each `NotRun` item must state whether it blocks task acceptance, the long-term integration branch, the stable branch, or only the release.
- When a safe test database, external account, or destructive-operation permission cannot be provided, define manual verification, risk acceptance, or a not-applicable rationale; do not turn dangerous environment operations into a default acceptance prerequisite.

## 3. Mandatory Section Structure

### 3.1 Document Metadata and Upstream Baseline

At minimum: status, version, background-document version, target platform or version, data baseline, reviewer, and update time.

### 3.2 Document Purpose

State what problem this document solves, which downstream documents it constrains, and what responsibilities it does not assume.

### 3.3 User Goals, Core Paths, and Success Criteria

Describe at least:

- Who the core user is;
- Where the user enters;
- The key operation sequence;
- What the first screen or core response must provide;
- How clicks, expansions, or follow-up actions continue;
- The perceivable speed, accuracy, and failure behavior for the user.

### 3.4 Background-Constraint Summary

Cite only the key facts from the approved background: existing data structure, scale, platform limits, cost, runtime windows, compatibility requirements, and parts that cannot change this cycle. Do not duplicate the entire background document.

### 3.5 Confirmed Business and Architecture Boundaries

Record high-level decisions that have already affected the requirements scope, for example:

- Document or data granularity;
- Business identity and uniqueness;
- System responsibility division;
- Read/write boundaries;
- Allowed eventual consistency;
- Compatibility boundaries for future extension.

These are requirement-level constraints, not implementation details.

### 3.6 Scope and Non-Scope

List separately:

- What must be delivered this cycle;
- What is explicitly not done this cycle;
- Content that serves only as future design input;
- Content depending on other iterations or manual stages.

### 3.7 Terminology and Business Caliber

Define potentially ambiguous items:

- Entity identity, status, and time;
- Conditions for a record to be valid or eligible for computation;
- Formulas for ordering, aggregation, discount, price, etc.;
- Semantics of null, zero, out-of-stock, disabled, deleted, etc.;
- Deterministic selection rule when values are equal;
- Whether UTC is used for time and how time zones are displayed.

### 3.8 Functional Requirements

Use stable IDs such as `FR-001`. Each item is recommended to use the following format:

```markdown
### FR-001: <Name>

**Goal**:

**Rules**:
- The system must …
- The system must not …

**Boundaries and exceptions**:

**Acceptance criteria**:
```

### 3.9 Special-Flow Requirements

When a feature includes synchronization, import, migration, batch processing, full build, incremental processing, or task scheduling, it should be numbered separately and explain:

- Trigger method;
- Input scope;
- Concurrency and mutual exclusion;
- Success, partial success, failure, timeout, and cancellation;
- Retry, recovery, and idempotency;
- How processed data is confirmed;
- Allowed latency and freshness target.

### 3.10 Non-Functional Requirements

Use IDs such as `NFR-001`, evaluating at least:

- Performance and response time;
- Capacity and growth;
- Availability and recovery;
- Consistency and data integrity;
- Security, permissions, and secret management;
- Observability;
- Compatibility;
- Testability;
- SOLID and module boundaries.

When no ready test environment exists, do not delete the relevant requirements; treat environment testing as subsequent manual acceptance, and explicitly state which stage it blocks, who executes it, and what authorization is needed.

### 3.11 Platform and External Limits

List specific versions, free quotas, protocols, connections, and security and capacity limits. Volatile information should cite official sources and verification dates.

### 3.12 Migration and Compatibility Requirements

Clarify:

- Whether it runs in parallel with the old feature;
- Whether shadow running or dual-read comparison is needed;
- Whether the existing data structure remains unchanged;
- What happens to users and data on rollback;
- Which cleanup work is deferred.

### 3.13 Acceptance Checklist

The acceptance checklist should cover structure, business function, sync/migration, exceptions, performance, and security/SOLID boundaries. Each item should be provable by automated test, static check, or manual environment verification.

Each key acceptance records at least:

| Field | Meaning |
|---|---|
| Acceptance ID | Stable ID, e.g. `AC-001` |
| Related requirement | `FR/NFR` ID |
| Acceptance level | Level 0/1/2/3, multi-select allowed |
| Environment & data | Minimal necessary environment, config, and sample |
| Execution responsibility | Developer, Reviewer, System Reviewer, project owner, etc. |
| Automation degree | Automatic, semi-automatic, manual |
| Blocking stage | Task, long-term integration branch, stable branch, or release |
| Success evidence | Test, log, metric, screenshot, data reconciliation, etc. |
| NotRun rule | Reason, risk, subsequent owner, and exception approver |

The acceptance level describes at which stage the business risk must be proven; specific test class names, scripts, or tools must not be hard-coded into requirements.

#### 3.13.1 Branch and Environment Strategy Reference

The requirements document should reference the logical branch roles configured by the project in the development-task document, but must not hard-code actual branch names. Default semantics:

- Level 0 blocks a single task from entering the iteration development branch;
- Level 1 blocks iteration from entering the long-term integration branch;
- Level 2 performs production-form system verification on the long-term integration branch;
- Level 3 and final merge review block entry into the stable branch.

The project owner may approve a different mapping, but must record the rationale, risk, and alternative gate.

### 3.14 Confirmed Decisions and Pending Items

- Confirmed decisions are recorded by sequence number with conclusion and impact scope.
- Pending items use checkboxes, confirmed item-by-item by the project owner.
- The document as a whole must not be set to "Approved" until pending items are cleared to zero.

## 4. Boundary Between Requirements and Implementation

Requirements may stipulate:

- A compatible platform or protocol version must be used;
- A certain database structure must remain unchanged;
- A certain user-visible capability or runtime mode must be supported;
- It must work under a certain capacity and latency;
- Permission boundaries or SOLID must be followed.

Requirements typically should not stipulate:

- Specific names of classes, interfaces, methods, and directories;
- A fixed number of items per batch or fixed page size;
- Specific SQL, DSL, or JSON mapping;
- Which internal design pattern to use;
- Which file the development task changes first.

If the project owner has explicitly specified an implementation restriction, it should be written as a "confirmed technical constraint" and expanded in the detailed design.

## 5. Traceability Requirements

The requirements document should maintain minimal traceability:

| Requirement ID | Source problem/goal | Acceptance method | Status |
|---|---|---|---|
| FR-001 | ISSUE-001 | Automated test / manual verification | Pending review |

Complex projects may expand in detailed design into "requirement → design section → development task → test evidence."

## 6. Execution-Agent (WorkBuddy / Codex) Execution Requirements

1. Use only the approved background as the fact baseline.
2. Help the user identify missing business rules and exception boundaries through dialogue.
3. Give candidate approaches, benefits, and costs for important trade-offs, and let the user decide.
4. Update decisions and review status immediately after the user confirms item-by-item.
5. Proactively identify scope expansion and put it into non-current-scope or pending items.
6. Do not change user-visible behavior for implementation convenience.
7. After each round of updates, list the items still needing review.
8. Help the project owner judge which level should prove each acceptance, but do not write technical test details as business requirements.
9. When an acceptance needs dangerous or unavailable environment operations, provide a safe alternative, manual verification, and risk-acceptance plan.

## 7. Prohibitions

- Writing large-scale requirements without a background baseline;
- Writing candidate approaches as confirmed requirements;
- Using technical jargon to mask unclear business caliber;
- Writing only the happy path, omitting no-data, failure, timeout, and partial-success;
- Miswriting future API design discussion as current-cycle API development scope;
- Adding deployment, go-live, or real-environment operations into the functional-requirement delivery scope;
- Writing only "automated test / manual verification" without stating level, blocking stage, and owner;
- Defaulting high-risk operations such as creating, deleting, or truncating a real database to acceptance conditions;
- Silently changing requirements in detailed design after requirements are approved.

## 8. Review Checklist

- [ ] User goals and core paths are complete;
- [ ] In-scope, out-of-scope, and future-input boundaries are clear;
- [ ] Data source of truth and consistency boundary are clear;
- [ ] Business terminology, formulas, filter, and ordering rules are unambiguous;
- [ ] All functional requirements are numbered and acceptable;
- [ ] Key acceptances have specified Level 0–3, owner, blocking stage, and success evidence;
- [ ] `NotRun` risk, subsequent responsibility, and exception-approval rule are clear;
- [ ] Sync/migration/batch exception semantics are complete;
- [ ] Performance, capacity, security, observability, and compatibility requirements are clear;
- [ ] Migration, parallel, and rollback requirements are clear;
- [ ] Each requirement traces to a background goal or problem;
- [ ] All important decisions are confirmed by the project owner;
- [ ] Pending items are zero.

## 9. Reusable Template

```markdown
# <Feature Name> Requirements Specification

> Document status: Draft
> Version: V0.1
> Upstream baseline: <background document and version>
> Target platform/version:
> Reviewer:
> Last updated:

## 1. Document Purpose
## 2. User Goals and Success Criteria
### 2.1 Core User Path
### 2.2 User-Perceivable Success Criteria
## 3. Background and Constraint Summary
## 4. Confirmed Business and Architecture Boundaries
## 5. Scope
### 5.1 In-Scope This Cycle
### 5.2 Out-of-Scope This Cycle
### 5.3 Future Design Input Boundary
## 6. Terminology and Data Caliber
## 7. Functional Requirements
## 8. Special-Flow Requirements
## 9. Non-Functional Requirements
## 10. Platform and External Limits
## 11. Migration and Compatibility
## 12. Requirement Traceability and Tiered Acceptance Checklist
### 12.1 Level 0–3 Acceptance Mapping
### 12.2 NotRun and Blocking Rules
## 13. Confirmed Decisions
## 14. Pending Items
```

## 10. Definition of Done

The requirements document may be marked "Approved" and used as the detailed-design baseline only when scope, caliber, exception boundaries, acceptance levels, blocking stages, and owners are all clear, and all pending items have been confirmed by the project owner.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.5 | 2026-08-20 | WorkBuddy | §6 title "Codex execution requirements" changed to "Execution-Agent (WorkBuddy/Codex) execution requirements," removing single-point binding |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
