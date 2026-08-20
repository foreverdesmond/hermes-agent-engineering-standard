# Background & Current-State Analysis Spec

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Document positioning: the first formal document in the development chain
> Output goal: form a credible fact baseline and problem definition, without prematurely committing to a specific implementation
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Originally finalized: 2026-08-11
> Revised: 2026-08-20
> Reviewer: Richy (approved)

## 1. Applicability

Before starting any of the following work, a background & current-state analysis should be written first:

- A new feature spans multiple modules, data tables, or external systems;
- An old feature is being upgraded or refactored;
- A search engine, database, middleware, cloud service, or third-party interface is being migrated;
- The current problem involves performance, capacity, consistency, or legacy technical debt;
- The user has raised an idea or concern that has not yet formed a stable requirement.

## 2. Questions the Document Must Answer

The background document must answer:

1. Who encountered what problem, in what scenario?
2. How does the current system actually work, rather than how it was originally intended to work?
3. What evidence do the code, database, configuration, and runtime environment each provide?
4. Does the problem affect correctness, experience, performance, cost, or maintainability?
5. What business, technical, time, resource, and environment constraints currently exist?
6. Which existing designs can be preserved?
7. Which content is fact, and which is only inference or a candidate approach?
8. Before entering the requirements stage, what issues still need confirmation by the project owner?
9. Which entry points, scopes, concurrency branches, and external resources will actually be activated at production runtime?
10. Which real dependencies are replaced by existing automated tests, and therefore cannot be proven by them?

## 3. Input and Investigation Methods

### 3.1 Must Read First

- Relevant entry points, services, domain models, infrastructure adapters, and tests;
- Database entities, mappings, migrations, SQL, and indexes;
- Current configuration structure and runtime mode;
- Production composition root, Host, startup entry, task scheduling, and runtime mode;
- Dependency-injection lifetimes, factories, Service Locator, static state, and shared resources;
- Existing design documents, historical decisions, and known issues;
- Logs, screenshots, and real business flows provided by the user.

### 3.2 Real-Environment Investigation Boundaries

- By default, perform read-only checks only.
- Before connecting to a real database, search engine, or cloud service, confirm user authorization and environment scope.
- Record query time, environment, data snapshot, and filter conditions.
- Do not modify production data, indexes, configuration, or task state merely to "verify."
- Sensitive connection strings, passwords, tokens, and personal information must not be written into the document.

### 3.3 Evidence Tiers

| Type | Writing requirement |
|---|---|
| Verified fact | Give the code location, query result, log, or official source |
| Measurement result | Record time, environment, sample range, and unit |
| Inference | Explicitly write "inference," and state the basis and uncertainty |
| Recommendation | Explicitly write "candidate recommendation"; must not be disguised as a confirmed requirement |
| Pending confirmation | State who decides and what that decision affects |

## 4. Mandatory Section Structure

### 4.1 Document Metadata

At minimum: status, version, investigation scope, data-snapshot time, upstream materials, reviewer, and update time.

### 4.2 Task Background and Business Goals

- User or operational flow;
- Current pain points;
- Why this is being addressed now;
- Business impact if not addressed;
- Directional description of the ideal outcome.

This section must not be written directly as a list of technical components.

### 4.3 Investigation Scope and Exclusions

- Which projects, modules, data, and environments were inspected;
- Which content was not inspected due to permission, environment, or time;
- The boundary of "analyze only, do not modify" for this round;
- The time range the conclusions apply to.

### 4.4 Current System and Data Flow

Explain:

- Entry points and triggers;
- Module responsibilities and dependencies;
- Where data is produced, transformed, stored, and consumed;
- Primary read paths and write paths;
- Data source of truth, cache, index, and derived data;
- External systems and manual operation points.

Beyond compile-time module relationships, the runtime-activated topology must also be investigated:

- Processes, Hosts, APIs, Jobs, commands, or other actual entry points;
- Which components are activated by different runtime modes and configuration combinations;
- Who creates and holds Singleton, Scoped, Transient, or other lifetime objects;
- Indirect resolution paths such as factories, `IServiceProvider`, reflection, plugins, `Lazy<T>`;
- Which state and resources are shared across parallel tasks, threads, requests, tenants, batches, or partitions;
- Ownership and release of database sessions, transactions, files, network connections, and external clients.

Projects that do not use a dependency-injection framework must still express object lifetime and resource ownership using their actual mechanism.

A minimal-necessary flow diagram should be used when the flow is complex.

### 4.5 Current Data Model and Scale

At minimum:

- Core entities and relationships;
- Primary keys, business uniqueness, and association methods;
- Key indexes and constraints;
- Current record count, growth expectation, and duplication multiplier;
- Data-quality conditions such as nulls, duplicates, and orphan data;
- Whether quantities are exact, sampled, or estimated.

### 4.6 Current Runtime and Update Mechanism

Describe per the actual situation of the feature:

- Triggers such as request, scheduled, event, manual, or process-end;
- Single-run, continuous, full/incremental, or other runtime modes;
- Actually-existing concurrency, failure, retry, and recovery methods;
- Logging, alerting, health checks, and manual intervention;
- Environment and deployment limits.

List at least the key differences between local development, automated testing, long-term integration, real external verification, and production environments. Branch names, environment names, and infrastructure are configured per project and must not be assumed in generic documents.

### 4.6.1 Current Test Capability and Proof Boundary

The actual coverage of existing tests must be investigated, not merely counted:

- Which tests use real production entry points and production registrations;
- Which dependencies are replaced by Mock, Fake, in-memory implementations, or hand-built stubs;
- Whether concurrency, lifetime, persistence, cancellation, partial failure, and resource release are covered;
- Which tests depend on a local database, container, network, or manual credentials;
- Which tests cannot be safely executed in the current environment;
- Which system risks remain even when existing tests are green.

Do not infer coverage level from the name "integration test exists"; the actually-activated path and stub boundary must be inspected.

Features not involving background execution or data updates are not forced to write irrelevant full/incremental/mutual-exclusion content.

### 4.7 Problem and Risk List

Organize by priority; recommended:

- `P0`: correctness, data loss, security, or production-blocking;
- `P1`: high-risk stability, performance, capacity, or consistency;
- `P2`: maintainability, extensibility, testing, and data quality;
- `P3`: experience optimization or low-risk technical debt.

Each item must at least contain:

```text
ID and title
Symptom
Evidence
Root cause or current inference
Impact
Trigger condition
Whether it needs to be addressed in this cycle
```

### 4.8 Preservable Existing Capabilities

Clarify which code, tables, flows, interfaces, or operational methods can continue to be used, and the reasons for keeping them. Background analysis should not list only shortcomings.

### 4.9 Constraints and Invariants

Including but not limited to:

- Data structures or compatibility that must not be broken;
- Available infrastructure and cost ceiling;
- Acceptable consistency latency;
- Maintenance windows and task-run cycles;
- Data security and permissions;
- Refactoring explicitly not handled in the current iteration.
- Whether automatic creation, deletion, or truncation of databases, schemas, storage volumes, and external resources is allowed or forbidden;
- Rework cost the project accepts and environment-operation risk it does not accept;
- Which real-environment operations can only be performed manually by the project owner.

### 4.10 Candidate Directions and Open Items

Candidate directions may be proposed, but must:

- Be listed separately, not mixed with facts;
- State benefits, costs, and unresolved issues;
- Not be written as a final architecture before review;
- List the issues requiring the user's item-by-item judgment.

## 5. Execution-Agent (WorkBuddy / Codex) Execution Requirements

When an execution agent (WorkBuddy / Codex) writes this type of document, it should:

1. Search code and files first, then form a judgment.
2. Attach code paths, query evidence, or log basis to key conclusions.
3. Verify volatile or high-risk external facts against official sources.
4. Explain the business impact of technical problems using real user paths.
5. Explicitly state what cannot be inspected, without fabricating facts.
6. When current state disagrees with the user's memory, show the discrepancy with evidence and ask the user to confirm.
7. Not implement fixes directly at this stage unless the user explicitly authorizes otherwise.
8. Must not treat Context L1 permanent context, historical context packages, or old designs as current facts; must sample or verify authoritative sources item by item.
9. Draw the minimal runtime-activated path for cross-module tasks, and mark the junction points still unverified.

## 6. Prohibitions

- Listening to oral accounts without checking available code and data;
- Treating historical design documents as current runtime facts;
- Writing a single query's elapsed time as a formal performance benchmark;
- Citing data volumes without recording time and sample range;
- Locking down an unreviewed technical solution at the background stage;
- Ignoring user experience or operational reality for the sake of architectural tidiness;
- Mixing in detailed task breakdown, deployment steps, or coding commitments.
- Investigating only the target file without checking its callers, production entry points, and shared resources;
- Inferring that real lifetime, database, or external environment are verified from Mock/Fake test results.

## 7. Review Checklist

- [ ] Business goals and user scenarios are clear;
- [ ] Investigation scope and uninvestigated scope are explicit;
- [ ] Current architecture and data flow come from actual code;
- [ ] Production entry points, runtime modes, lifetimes, concurrency, and resource ownership have been investigated;
- [ ] Current test capability, stubs, and proof gaps have been recorded;
- [ ] Core data volumes have time, environment, and caliber;
- [ ] Problems are prioritized and evidenced;
- [ ] Facts, inferences, recommendations, and open items are separated;
- [ ] Preservable capabilities and unbreakable constraints are listed;
- [ ] No unauthorized external modification was performed;
- [ ] Questions needed to enter the requirements stage are substantially answered;
- [ ] The project owner has confirmed the background baseline.

## 8. Reusable Template

```markdown
# <Feature Name> Background & Current-State Analysis

> Document status: Draft
> Version: V0.1
> Investigation scope:
> Data-snapshot time:
> Reviewer:
> Last updated:

## 1. Background and Business Goals
## 2. Investigation Scope, Method, and Limits
## 3. Current System Architecture and Data Flow
## 4. Current Data Model and Scale
## 5. Current Runtime and Update Mechanism
## 6. Runtime-Activated Topology, Lifetime, and Shared Resources
## 7. Current Test Capability and Proof Boundary
## 8. Problem and Risk List
### P0
### P1
### P2
## 9. Preservable Existing Capabilities
## 10. Constraints and Invariants
## 11. Candidate Directions (not yet confirmed)
## 12. Items Pending Review and Confirmation
## 13. Evidence and References
```

## 9. Definition of Done

This file may be marked "Approved" only when the project owner has confirmed that "the fact baseline is sufficient to support requirements discussion," the runtime topology and test-proof boundary are clear enough, and key questions no longer rely on unverified guesses. Candidate solutions need not be finalized at the background stage.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.5 | 2026-08-20 | WorkBuddy | §5 title and body "Codex execution requirements" changed to "Execution-Agent (WorkBuddy/Codex) execution requirements," removing single-point binding |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
