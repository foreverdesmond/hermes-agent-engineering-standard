# Detailed Design Spec

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Document positioning: transforms the reviewed requirements into an implementable, testable technical solution
> Prerequisite gate: the Requirements Specification has been approved overall
> Applicability: API, crawler, background task, data processing, search, migration, frontend, and other feature development
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Originally finalized: 2026-08-11
> Revised: 2026-08-20
> Reviewer: Richy (approved)

## 1. Document Goal

The detailed design must let a developer implement the feature correctly without relying on chat memory, and must let a reviewer judge whether the solution satisfies requirements, architectural principles, reliability, security, and testability.

The detailed design consists of two parts:

1. **Generic design skeleton**: must be covered by any feature;
2. **Specialized design modules**: selected on demand by feature type and risk.

Specialized modules that do not apply should not be forced into the body. If something seems relevant but ultimately does not apply, mark it briefly as `N/A` with a reason.

## 2. Boundary Between Requirements and Design

### 2.1 What the Requirements Document Owns

The requirements document owns:

- What result the user or business needs;
- Business terminology, rules, and acceptance caliber;
- In-scope and out-of-scope this cycle;
- External constraints such as performance, security, and compatibility;
- Which important trade-offs have been confirmed by the project owner.

### 2.2 What the Detailed Design Owns

The detailed design owns:

- Which components in the existing system implement the requirements;
- How components collaborate;
- How data and control flow;
- How interfaces, models, and external dependencies are organized;
- How normal, exceptional, and boundary cases are implemented;
- How to test and prove the implementation is correct.

The detailed design may reference business rules but must not reinvent or modify them. When a requirement is found ambiguous, return to the requirements document rather than silently making a business decision in the design.

## 3. Feature Classification Before Design

Before writing the detailed design, first judge which types this feature involves. A feature may belong to multiple types.

| Feature type | Typical example | Specialized modules usually needed |
|---|---|---|
| API query | product list, detail, search interface | API, query & cache, performance, security |
| API write | create, modify, delete, status change | API, transaction & concurrency, idempotency, security |
| Crawler / data collection | supermarket product & price crawling | crawler, external integration, scheduling, data quality |
| Background task | scheduled cleanup, sync, stats, notification | Job, state & recovery, observability |
| Data migration / sync | DB migration, search index rebuild | data processing, full/incremental, verification & switch |
| Search feature | full-text, suggestion, highlight, ranking | search & index, query, capacity |
| Domain rules | pricing, discount, eligibility | business-rule implementation, precision & determinism |
| Frontend / UI | page, component, interaction flow | UI state, interaction, accessibility, interface contract |
| External service integration | payment, map, AI, email | external contract, rate limiting, failure & degradation |
| Infrastructure | logging, JobHost, cache, storage adapter | module boundary, config, security, compatibility |

The document's first page should list the selected feature types and specialized modules, so the reviewer knows what must be checked and what is explicitly not applicable.

## 4. Generic Design Skeleton (must be covered by all features)

### 4.1 Document Metadata

At minimum:

- Document status and version;
- Approved requirements document and version;
- Feature types of this cycle;
- Selected specialized design modules;
- Owner, reviewer, and last-updated time;
- Target runtime environment or compatible version (if applicable).

### 4.2 Design Goal and Conclusion

In concise text state:

- What this design intends to solve;
- The overall implementation approach finally chosen;
- The most critical components and boundaries;
- The main impact on the existing system;
- What this design explicitly does not handle.

### 4.3 Requirements Traceability

Every approved requirement must have a design landing point:

| Requirement ID | Design landing point | Verification method | Status |
|---|---|---|---|
| FR-001 | 5.2, 6.1 | unit test, interface test | Covered |

If a requirement needs no code change, also state the evidence that it is satisfied by existing capability.

### 4.4 System Context and Impact Scope

State:

- Where the feature is triggered;
- Who calls it, and what it depends on;
- Where input comes from and output goes to;
- Which projects, modules, or data it modifies;
- Which existing interfaces and behaviors must stay compatible;
- Which adjacent systems are out of scope this cycle.

Use architecture, flow, or sequence diagrams only when the relationship is genuinely complex; do not manufacture diagrams for simple features.

### 4.5 Component Responsibility and Dependency Direction

For new or changed components state:

- Single responsibility;
- Capabilities exposed outward;
- Abstract dependencies;
- Where external technical details are adapted;
- Project and directory placement;
- Runtime call direction;
- Compile-time reference direction.

The runtime call direction and the compile-time dependency direction must be described separately; a single catch-all layering arrow cannot substitute for both.

#### 4.5.1 Runtime Activation, Lifetime, and Resource Ownership

When dependency injection, concurrency, background execution, persistence, external clients, or mutable state are involved, the runtime activation path must be given, not just project reference relationships.

Record at least:

| Object or resource | Creation entry | Lifetime/scope | Mutable or thread-safe? | Concurrent callers | Owner & release responsibility |
|---|---|---|---|---|---|

The design must check:

- Whether long-lived objects hold short-lived objects directly or indirectly;
- Whether factories, Service Locator, reflection, plugins, `Lazy<T>`, callbacks, and static fields hide real dependencies;
- Whether requests, tasks, tenants, batches, partitions, or other parallel units need an independent Scope;
- Whether database sessions, transactions, Writers, Streams, locks, caches, and external clients can be shared concurrently;
- Who creates, commits, rolls back, cancels, and releases resources, and whether exception paths are consistent;
- Whether runtime mode or config combination changes the object graph and parallel relationships.

Projects without a DI framework must still complete this section using their actual object-creation mechanism.

### 4.6 Core Interface, Model, and Contract

Design only what this feature actually needs to add or change:

- Input, output, and return status;
- Required, optional, default values, and validation;
- Cancellation, timeout, and error semantics;
- Model's layer of belonging;
- Versioning and compatibility approach;
- Whether external SDK types may cross the boundary.

Database entities, domain models, application DTOs, API contracts, and external-system models are not the same concept and should be placed by responsibility.

### 4.7 Main Execution Flow

Describe at least:

- Main success path;
- Key branches;
- No-data or empty result;
- Invalid input;
- Dependency failure;
- Cancellation and timeout;
- Necessary recovery or degradation.

A simple synchronous call can use step descriptions; use a sequence diagram or state machine only when multiple components, async state, or loop processing exist.

### 4.8 Error Handling and Reliability

Define by feature risk:

- Error classification;
- Which errors are returned to the caller;
- Which errors can be retried;
- Retry count, backoff, and stop condition;
- How partial success is expressed;
- Whether idempotency, compensation, or recovery is needed;
- Whether degradation behavior masks real failures.

Not every feature needs a complex state machine, but every feature must state what happens on failure.

Exception ownership must also be defined: which layer catches, transforms, logs, and decides runtime status. Empty `catch`, "log-then-continue," returning an empty collection or default success as fault tolerance must have an approved business semantics, observability, and test; they must not be the default way to avoid failure.

### 4.9 Configuration and Security

State applicable content:

- Config items, default values, and startup validation;
- Environment differences;
- Authentication and authorization;
- Input validation and output protection;
- Source of passwords, tokens, and connection info;
- Sensitive-information protection in logs;
- Network protocol and least privilege.

Secrets must not enter code, ordinary config examples, or design documents.

### 4.10 Observability

State how to judge whether the feature is healthy:

- Key structured logs;
- Success, failure, and duration metrics;
- Necessary business counters or progress;
- Health status or diagnostic entry;
- Identifier correlating one request or task;
- What must not be logged.

Observability content should match the feature type. A plain API needs no batch cursor; a simple tool need not design a full monitoring platform.

### 4.11 Performance and Resource Impact

State per requirements:

- Main performance path;
- Expected data volume, request volume, or task frequency;
- CPU, memory, network, database, or external-service impact;
- Cache, pagination, concurrency, or batch parameters;
- Parameter basis and subsequent tuning method.

Low-risk features may briefly state "no significant new resource consumption" with the basis. Key parameters must not be hard-coded without basis.

### 4.12 Test Design

Plan by applicability:

- Unit test;
- Interface or contract test;
- Component integration test;
- End-to-end test;
- Performance and capacity test;
- Security test;
- Subsequent manual environment verification.

When a test environment is unavailable, record `NotRun`, reason, owner, and follow-up verification method; do not claim it passed.

When a database, persistence, real external system, or production-form entry is involved, a safe verification path must also be designed:

- Clarify which verifications can run offline on the task branch and which can only run on the iteration candidate or real environment;
- Prefer verification entries that are network-free, read-only, Fake, rollback-able, or never reach a write point, while honestly stating their proof boundary;
- Without an isolated test database, real writes must not be the default acceptance means for the developer or Reviewer;
- When the project owner must manually start, observe, and promptly stop, the stop point, estimated first-side-effect point, log criteria, and residual risk must be given;
- Which task or role bears development, Review, integration, and real-environment acceptance must be explicit.

Test design must reference the Level 0–3 acceptance mapping in requirements, and state the proof boundary for each stub:

| Test or level | Real path | Replaced dependency | Can prove | Cannot prove |
|---|---|---|---|---|

Unit, Mock, Fake, or in-memory tests alone cannot prove the production object graph, real concurrency resources, database Provider behavior, or real external system are correct. The design should place residual risk at the appropriate level, not hide it.

#### 4.12.1 Production Composition-Root Risk Assessment

When adding or modifying production registration, service lifetime, key config contract, Host/startup entry, or runtime key-entry activation, the production composition-root integration test must be assessed and planned. Merely adding not-yet-wired components, test-only code, or local construction tweaks that do not affect the production object graph does not automatically trigger a full composition-root test.

Test design must be explicit:

- Use the same service-registration extension or Host build path as production; do not write an equivalent registration just for the test;
- Under a valid minimal config, resolve all affected key entry services one by one; where applicable, enable `ValidateOnBuild` and `ValidateScopes` together;
- If a service is activated only at Host startup or first run, the test must cover that activation point, not merely verify the container can be created;
- Startup config validation, Options' `ValidateOnStart`, hand-`new` components, or using Mock to bypass the container can only be supplementary tests, not sufficient evidence that the production dependency graph is resolvable;
- For these common wiring errors, give a check caliber: whether `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>` and direct injection of `T` match the actual registration; whether named Options' name and consumption method match; whether Singleton/Scoped/Transient dependency direction and Scope are valid; whether open generics can close and resolve; whether factory registration can create the target service with production parameters and its return type, lifetime, and internal dependencies are correct;
- For Service Locator, factory, `Lazy<T>`, static cache, and indirect resolution in callbacks, perform actual activation and concurrency checks; do not rely only on static container validation;
- If the test host lacks logs, database, clock, or external services, explicit test doubles, test containers, or network-free config may be used, but the production registration path under review must not be replaced or bypassed; test results must distinguish "test host lacks necessary support" from "production DI registration or dependency-graph defect";
- Evidence must at least include test name, real registration entry, list of resolved key services, config environment or stub description, execution command, and result.

Counter-example: Options binding & validation tests pass and the component can be hand-constructed, but that does not mean the production container can activate the key entry service consuming that component. If the consumer requests a type, name, or lifetime inconsistent with the composition root's actual registration, the production composition-root integration test must catch it at the development stage.

### 4.13 Compatibility, Migration, and Rollback

Every feature should judge whether this section is needed:

- When there is no compatibility or migration impact, write `N/A` with a reason;
- When modifying an existing interface, data, config, or runtime mode, state the upgrade order;
- When parallel running, canary, data conversion, or rollback is needed, select the corresponding specialized module to expand;
- The timing of cleaning old code or old resources must be explicit.

### 4.14 Design Decisions, Risks, and Pending Items

Record:

- Main candidate approaches;
- Final choice and rationale;
- Rejected approaches;
- Remaining risks;
- Conflicts needing the project owner's decision;
- Out-of-scope this cycle.

Each remaining risk must also state at which Level it is verified, which branch role or release stage it blocks, and who accepts the risk when the environment is insufficient.

Important decisions use `DEC-001`; SOLID conflicts may use `SOLID-DEC-001`.

## 5. SOLID Hard Constraints

All detailed designs must review item by item, but the review content should fit the current feature rather than mechanically count interface quantities.

| Principle | Question that must be answered |
|---|---|
| S: Single Responsibility | Does the new component have only one primary reason to change? |
| O: Open/Closed | Can a reasonable new implementation be done by extension, or must the stable core be modified every time? |
| L: Liskov Substitution | Do implementations of the same abstraction keep consistent input, output, error, and cancellation semantics? |
| I: Interface Segregation | Does the caller depend only on the minimal capability it actually uses? |
| D: Dependency Inversion | Does the core rule depend on abstractions, with external technical details outside the boundary? |

Concrete conclusions related to this feature must be written. Old code may be left unrefactored for now, but new code must not copy its violating design without reason.

On conflict:

```markdown
### SOLID-DEC-001: <conflict title>

- Conflict:
- Approach A:
- Approach B:
- Impact:
- Recommendation:
- Project owner decision: pending review
```

## 6. Specialized Design Modules

The following modules are a design checklist, not fixed sections of every detailed design. Select only modules relevant to the current feature.

### 6.1 API and Service Interface

For query or write APIs:

- Route, method, and version;
- Request, response, and error contract;
- Parameter validation;
- Authentication, authorization, and data scope;
- Pagination, sorting, filtering, and caching;
- Write idempotency, concurrent update, and conflict;
- Rate limiting, timeout, and compatibility.

### 6.2 Crawler and Data Collection

For crawlers and external data fetching:

- Data source, entry page, and crawl scope;
- Schedule frequency and startup method;
- Request rate limiting, concurrency, timeout, and retry;
- Page or interface parsing boundary;
- Data cleaning, deduplication, and identity judgment;
- Incremental discovery, checkpoint, and resumable recovery;
- Data-quality validation;
- Source change, ban, or partial-failure handling;
- Compliance, terms of service, and sensitive-information boundary.

### 6.3 Background Task and Job

For scheduled or manual background tasks:

- Trigger and parameters;
- Single run or resident run;
- Reentrancy, mutual exclusion, and concurrency scope;
- Max run time, cancellation, and process exit;
- Run state, progress, and recovery;
- Exception capture and exit code;
- Responsibility boundary between scheduler and business service.

### 6.4 Data Storage and Persistence

Select only when the feature adds or changes data persistence:

- Tables, columns, relationships, and constraints;
- Primary key, business key, and uniqueness;
- Transaction boundary and isolation;
- Index and query plan;
- Data lifetime, deletion, and audit;
- Idempotency and upgrade order of DB scripts.

### 6.5 Data Processing, Sync, and Migration

Only for derived data, import, sync, or migration:

- Source, target, and data source of truth;
- Transformation, filtering, aggregation, and validation;
- Change capture;
- Full, incremental, and compensation;
- Concurrency, idempotency, and unknown-result;
- Switch, rollback, and old-data cleanup;
- Reconciliation and consistency window.

### 6.6 Search and Index

Only for full-text search or search storage:

- Document granularity and document identity;
- mapping, analyzer, and field usage;
- Query, filter, sort, highlight, and suggestion;
- Pagination and relevance;
- Index lifecycle and alias;
- Capacity, sharding, and bulk write.

### 6.7 Query, Cache, and Read Optimization

For read-heavy features:

- Query shape and return granularity;
- Data loading and N+1 risk;
- Index, cache, and invalidation;
- Pagination and large result sets;
- Consistency and stale data;
- Hotspot and degradation.

### 6.8 Domain Rules and Computation

For complex business computation:

- Input and preconditions;
- Rule execution order;
- Precision, rounding, and unit;
- Deterministic selection on equal values;
- Time and time zone;
- Rule extension method;
- Unit-testable pure-computation boundary.

The business rules themselves must come from the requirements document; this module only explains their technical implementation.

### 6.9 Transaction, Concurrency, and Consistency

Expand only when shared writes or multi-step state changes exist:

- Transaction boundary;
- Optimistic or pessimistic concurrency;
- Lock, lease, and unique constraint;
- Duplicate request and idempotency key;
- Multi-store consistency;
- Compensation and eventual consistency.

### 6.10 External System Integration

- External contract and version;
- Client responsibility;
- Authentication and connection;
- Rate limiting, retry, and circuit breaking;
- Error mapping;
- Mock, contract test, and behavior when unavailable;
- Isolation of external SDK from core code.

### 6.11 Frontend and Interaction

- Page or component responsibility;
- User interaction and state change;
- Loading, empty, error, and retry states;
- Form validation;
- Responsive layout and accessibility;
- Client cache and state management;
- API contract and compatibility.

## 7. Specialized Module Selection Rules

### 7.1 When Selection Is Mandatory

- The dimension directly carries a requirement;
- The dimension will add or change code, table, config, or external resource;
- The dimension contains obvious correctness, security, performance, or operations risk;
- Not stating it would let different developers make incompatible implementations.

### 7.2 When Selection May Be Skipped

- This feature is entirely unrelated to the dimension;
- The existing design is reused as-is with a clear reference;
- It is only a long-term idea, not in current scope.

### 7.3 Prohibited Practices

- Forcing an unrelated database, messaging, cache, or index just because the template exists;
- Writing all optional modules as empty headings;
- Using `N/A` to skip an actually-existing high-risk problem;
- Promoting one migration project's specific technical solution into a project-wide general rule.

## 8. Execution-Agent (WorkBuddy / Codex) Execution Requirements

1. Before starting, confirm the requirements version and overall review status.
2. Inspect existing code layering, entry points, and dependencies; do not design project structure from nothing.
3. Complete "feature classification and specialized-module selection" first, then expand the body.
4. Keep the design simple for simple features; expand necessary details for high-risk features.
5. Compare main alternatives for key approaches; do not manufacture formalism for obvious small implementations.
6. Proactively perform SOLID review, but do not equate SOLID with "more modules is better."
7. Use official sources for the target version of external-platform facts.
8. When a business-rule or scope change is needed, return to the requirements stage for a ruling.
9. After each round of updates, list new decisions, selected modules, and remaining pending items.

## 9. Generic Review Checklist

- [ ] Feature-type and specialized-module selection is reasonable;
- [ ] All requirements have a design landing point;
- [ ] System context and impact scope are clear;
- [ ] Component responsibility, runtime call, and compile-time dependency are clear;
- [ ] Runtime activation path, lifetime, indirect dependency, and resource ownership are clear;
- [ ] Interfaces, models, and contracts are sufficient to guide implementation;
- [ ] Main path, boundaries, and failure behavior are clear;
- [ ] Configuration, security, and secret boundaries are clear;
- [ ] Log, metric, and diagnostic methods match the feature;
- [ ] Performance and resource impact are assessed by risk;
- [ ] Automated and manual test design is complete;
- [ ] Database, external system, and production entry have a clear no-side-effect verification path or an approved `NotRun`/manual-verification plan;
- [ ] What test stubs can and cannot prove is explicit;
- [ ] When the production object graph or key entry changes substantially, a real production-registration-path composition-root test is planned; risk basis is given when not triggered;
- [ ] Compatibility, migration, and rollback are handled or explicitly not applicable;
- [ ] The five SOLID items have concrete conclusions;
- [ ] Important approaches and conflicts are confirmed by the project owner;
- [ ] Inapplicable modules are not force-fitted;
- [ ] Pending items are zero.

## 10. Reusable Template

```markdown
# <Feature Name> Detailed Design

> Document status: Draft
> Version: V0.1
> Requirements baseline: <requirements document and version>
> Feature type: API query / crawler / Job / …
> Selected specialized modules: 6.1, 6.7, 6.9
> Reviewer:
> Last updated:

## 1. Design Goal and Conclusion
## 2. Requirements Traceability
## 3. System Context and Impact Scope
## 4. Component Responsibility and Dependency Direction
## 5. Runtime Activation, Lifetime, and Resource Ownership
## 6. Core Interface, Model, and Contract
## 7. Main Execution Flow
## 8. Error Handling, Exception Ownership, and Reliability
## 9. Configuration and Security
## 10. Observability
## 11. Performance and Resource Impact
## 12. Tiered Test Design and Stub Proof Boundary
## 13. Compatibility, Migration, and Rollback
## 14. Specialized Design
### <only the specialized modules selected for this feature>
## 15. Design Decisions, Residual Risk, and Out-of-Scope
## 16. Requirements Traceability and Review Checklist
## 17. References
```

## 11. Minimal Examples by Feature Type

### 11.1 API Query/Update

Recommended: API and Service Interface, Query & Cache, Transaction & Concurrency (when writes exist), Security, Performance.

Usually not needed: crawler, search mapping, full/incremental migration.

### 11.2 Crawler Feature

Recommended: Crawler & Data Collection, Background Task, External System Integration, Data Persistence, Data Quality, Observability.

Usually not needed: API routing, search mapping, alias switch.

### 11.3 Search Engine Migration

Recommended: Data Processing & Migration, Search & Index, Background Task, External System Integration, Transaction & Concurrency, Verification & Rollback.

### 11.4 Simple Domain-Rule Change

Recommended: Domain Rules & Computation, affected interface or model, unit test, compatibility.

Usually not needed: complex architecture diagrams, runtime-state tables, migration flow, or capacity model.

## 12. Definition of Done

The detailed design may be marked "Approved" only when the design content commensurate with the selected development track is complete, all requirements have a design landing point, runtime lifetime and resource ownership are clear, exception ownership is explicit, the stub proof boundary and Level 0–3 verification landing points are executable, the production composition-root test is planned by risk, the five SOLID items are judged and conflicts ruled, and pending items are cleared to zero.

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 approved baseline |
| V2.5 | 2026-08-20 | WorkBuddy | §8 title "Codex execution requirements" changed to "Execution-Agent (WorkBuddy/Codex) execution requirements," removing single-point binding |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as official V2.5 baseline |
