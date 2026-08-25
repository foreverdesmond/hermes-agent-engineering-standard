# AI-Assisted and Multi-Agent Software Development Standard

<div align="center">

**Moving AI-assisted development from “able to write code” to “able to deliver software in a controlled manner”**

A software engineering standard for AI-assisted development and multi-Agent collaboration

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://choosealicense.com/licenses/mit/)
[![X: @Richyisaflower](https://img.shields.io/badge/X-@Richyisaflower-black?logo=x)](https://x.com/Richyisaflower)

</div>

## What Is It?

This is a software engineering standard for AI-assisted development and multi-Agent collaboration.

Its goal is to transform an incompletely defined development objective into a software change that can be reviewed, implemented, verified, merged, and traced.

It is not tied to any particular programming language, technology framework, database, code-hosting method, or AI model. Whether a single person is developing with the assistance of AI or multiple Agents are collaborating in parallel, an appropriate process can be selected according to task risk.

What the product should do is always decided by humans; the specific investigation, design, implementation, and verification are completed collaboratively by Agents with clearly defined responsibilities. When using AI to develop software, the following must be controlled above all else:

> When using AI to develop software, how to control the development scope, responsibility boundaries, execution permissions, verification evidence, and final quality.

## Why Is It Needed?

AI can already investigate code, generate implementations, write tests, and complete a substantial amount of engineering work quickly. However, “code generation completed” does not mean that a “software change is trustworthy.” In real AI-assisted and multi-Agent development, the following problems tend to recur:

| Risk | Typical manifestation |
| --- | --- |
| Fact drift | An Agent relies on outdated documents, expired context, or another Agent’s summary without rechecking the current code and runtime facts. |
| False completion | Passing unit tests and a successfully compiling project are taken to mean that the system is correct. |
| Excessive reliance on context | The Context Package is treated as the facts themselves; although the code has changed, implementation still follows the old map. |
| Self-review and self-approval | The same party writes the code, reviews the code, and performs the merge. |
| Unsafe execution | For testing convenience, an Agent creates, deletes, or clears a database or other real resources without authorization. |
| Irrecoverable state | After a task is interrupted, its progress, evidence, and next action cannot be determined accurately, and the process cannot be audited. |
| Design deviation | The implementation has deviated from the requirements or design, but the deviation is concealed by green test results. |

In single-person development, these risks are often controlled through the developer’s experience, memory, and continuous vigilance. Once multiple Agents work in parallel, it is difficult to maintain the same level of control through individual attention alone. This control capability must therefore be built into roles, documents, evidence, and gates.

This standard comes from real AI-assisted development practice. Problems encountered during development were reviewed and organized into reusable engineering rules.

## What Problems Does It Solve?

This standard converts common risks into executable control mechanisms:

| Problem to solve | Corresponding mechanism |
| --- | --- |
| Not knowing what the current facts are | Current code, configuration, runtime entry points, and reviewable evidence take precedence; documents and context serve only as controlled navigation. |
| Not knowing whether a task is truly complete | Verification Levels 0–3 distinguish local correctness, integration correctness, system correctness, and correctness in the real environment. |
| No one independently checks the result | Separate the responsibilities for requirements and design, implementation, review, integration, verification, and scheduling. |
| The Agent’s operating scope is uncontrollable | Define the modification scope, read scope, external resource boundaries, and human authorization boundaries. |
| Task state is easily lost | Use persistent, recoverable, and auditable task ledgers and evidence states. |
| The process is too heavy or too light | Select a lightweight, standard, or high-risk route according to risk. |
| The code passes but the design is wrong | Include requirements consistency, design consistency, and SOLID principles in the quality gates. |

## How Humans and Agents Divide the Work

Humans do not need to manage every line of code. They should focus on the judgments that genuinely require human responsibility.

Humans are primarily responsible for:

- Defining product objectives, business priorities, and success criteria;
- Confirming what is and is not included in the current task;
- Defining data, permission, security, and external-operation boundaries;
- Determining whether risks are acceptable and which matters require separate authorization;
- Reviewing evidence at key gates and deciding whether to proceed to the next stage;
- Approving merges, releases, and other operations with business or external impact.

Agents are primarily responsible for:

- Investigating the current code, configuration, runtime paths, and related facts;
- Expanding confirmed objectives into requirements, design, and development tasks;
- Writing, modifying, and verifying code;
- Performing independent review, integration, and tiered verification;
- Recording state, evidence, risks, and unfinished matters;
- Advancing tasks within the prescribed permissions and responsibility boundaries.

Humans still need to understand software quality, but they do not need to undertake every implementation detail. More important judgments include whether the requirements are correct, the scope is clear, the boundaries are closed, the evidence is sufficient, the risks are controllable, and the final product is worth delivering.

## The Six Core Roles

These six roles correspond to the division of responsibilities in a traditional software development process:

```text
Requirements and Design → Implementation → Independent Review → Integration → Verification → Scheduling and Coordination
```

| Role | Primary responsibility |
| --- | --- |
| Doc/Design Reviewer | Producing context documents and writing and reviewing various documents. |
| Implementer | Completing implementation, rework, and local verification in an independent task branch. |
| Reviewer | Performing an independent review based on accurate code diffs, commits, and evidence. |
| Integrator | Merging precise, reviewed commits and handling integration conflicts. |
| Validator | Performing post-merge integration verification, system verification, and regression verification. |
| Coordinator | Handling task dispatch, state management, evidence consumption, pausing, resuming, and gate coordination. |

Roles may be streamlined or combined to an appropriate degree according to task risk. However, a developer may not approve their own task, a merger may not replace independent review, and final business approval may not be completed by an Agent alone.

The current project uses Hermes as one implementation of the Coordinator. The standard defines scheduling responsibilities, state boundaries, and control requirements; other projects may use different scheduling implementations.

## How to Use This Standard

When using this standard, first define the problem clearly together with AI, then have Agents in the corresponding roles complete the engineering elaboration and implementation.

### 1. Discuss the Objective with AI and Form the Background and Requirements

Starting from an incompletely defined idea, discuss the following with AI:

- Why should this be done?
- What is the current system, business context, or usage scenario?
- What result does the user want?
- What problem needs to be solved this time?
- Which content is explicitly outside the current scope?
- What constraints, risks, and external dependencies exist?
- How will it ultimately be determined that the work is complete?

AI can help ask questions, investigate, organize facts, identify contradictions, and find omissions, producing:

- A task background and current-state analysis document;
- A requirements specification document.

These two documents should be continuously refined during discussion, investigation, and confirmation, with their content converging as facts and decisions are confirmed.

### 2. Close the Requirements Boundaries

Before entering design and coding, close the key boundaries of the current task as far as possible, including:

- Functional and non-functional scope;
- User behavior, system behavior, and exception handling;
- Input, output, data, and permission boundaries;
- Security boundaries and operations that cannot be automated;
- The scope of operations on external systems and persistent resources;
- Differences in behavior across environments;
- Acceptance methods and verification levels;
- Confirmed decisions, matters pending confirmation, and problems explicitly not being solved.

“Closing the boundaries” does not mean that all unknown information must disappear. Content that cannot be confirmed must be recorded as an unknown, assumption, risk, or matter pending authorization. An Agent may not fill it in as a fact or requirement on its own during a subsequent stage.

Only after the background and requirements documents are sufficiently clear, and the scope, acceptance criteria, and security boundaries have been confirmed, should the work enter the engineering elaboration stage.

### 3. Have the Corresponding Agents Complete the Engineering Elaboration

After the background and requirements boundaries have been confirmed, Agents in the corresponding roles continue with:

- Detailed design;
- Development task decomposition;
- Task dependencies and interface responsibilities;
- File and resource ownership;
- Context Packages and task delegation information;
- Verification plans and evidence requirements.

Humans do not need to write every technical implementation detail, but they do need to confirm that the design and task decomposition still conform to the approved objectives and boundaries, that no unauthorized new requirements have been introduced, and that each task has clear completion conditions.

### 4. Enter the Coding Stage After Task Decomposition Is Complete

Before coding begins, the requirements, design, and development tasks must have formed an executable task chain. Each task should have a clear scope, responsibility, dependency, verification method, and completion condition.

The roles then proceed according to their responsibilities:

```text
Implementer: implementation
  → Reviewer: independent review
  → Integrator: integration
  → Validator: tiered verification
  → Human: authorization decisions at key gates
```

Humans do not need to intervene continuously in every line of code, but they must make judgments at key points such as requirements confirmation, boundary closure, design approval, risk authorization, verification conclusions, and merge and release.

## Scope of Application and Iteration Boundaries

This standard is not suitable for planning, decomposing, and driving an entire large-scale project in one pass.

Large projects usually contain long-term evolving business objectives, continuously changing technical constraints, and substantial uncertainty that has not yet surfaced. If an Agent is asked to complete the requirements, design, and task decomposition for the entire project at the outset, this can easily produce expired context, over-design, and task chains that are difficult to maintain.

The best practice for this standard is to apply it to a clearly bounded, independently acceptable set of development tasks corresponding to one Sprint in a traditional Agile development process:

1. Select a clear iteration objective;
2. Complete the background investigation and requirements confirmation together with AI;
3. Close the scope, risk, and security boundaries for the iteration;
4. Have Agents complete the design and development task decomposition;
5. Enter coding after independent review;
6. Complete tiered verification, integration, and merging;
7. Determine the objective of the next iteration based on the actual results of this iteration.

Large products should be formed through multiple incremental iterations. Each iteration begins with a clear Sprint objective:

```text
Product vision
  → Sprint objective
  → Background and requirements confirmation
  → Design and task decomposition
  → Coding and independent review
  → Integration and tiered verification
  → Deliverable increment
  → Retrospective and entry into the next Sprint
```

Each iteration should use the current real code, runtime state, and verified results as its new factual baseline. Assumptions and context from the previous iteration may be reused only after being rechecked.

This standard serves the incremental delivery of large projects and should be used repeatedly in each Sprint.

## Verification: From Local Correctness to Real Deliverability

Test results must be interpreted together with the verification scope and runtime environment. This standard divides verification into multiple levels:

| Level | Question to answer |
| --- | --- |
| Level 0 | Is the task itself correctly implemented according to the requirements and design? |
| Level 1 | After multiple tasks are combined, is the integration behavior within the iteration correct? |
| Level 2 | When the system runs in its production form, are the critical paths correct? |
| Level 3 | In the real external environment, do the version-level business results meet the requirements? |

> Passing unit tests can only prove that local behavior satisfies the current tests; it cannot automatically prove that the system wiring, real environment, and final business results are correct.

If a verification level is not applicable, the reason, risk, and subsequent responsible person must be recorded. Unverified content may not be assumed to be correct.

## Select Process Intensity According to Risk

This standard does not require every task to follow a process of the same complexity. Projects should select an appropriate route according to the task’s scope of impact, reversibility, external dependencies, and cost of failure:

| Route | Applicable scenario | Typical requirements |
| --- | --- | --- |
| Lightweight | Local, reversible, low-impact changes with sufficient automation | Task description, targeted verification, and an independent second-perspective review. |
| Standard | Changes requiring design or integration, or affecting a small number of modules | Background and requirements, development tasks, Level 0, and applicable integration verification. |
| High-risk | Cross-module, multi-Agent parallel, critical-data, security, migration, or externally uncertain tasks | A complete document chain, Context Package, independent roles, Levels 0–3, and strict authorization. |

The process may be simplified according to risk, but factual accuracy, independent review, design consistency, and the safety of external operations may not be omitted.

## What Does the Standard Consist Of?

Each document is responsible for one role in the development process. The README provides an entry point and does not duplicate the complete gates:

| Document | Primary responsibility |
| --- | --- |
| [01-Background Analysis](./specs/01-background-analysis.md) | Establishing the current facts, runtime topology, evidence, and risk baseline. |
| [02-Requirements Specification](./specs/02-requirements-spec.md) | Defining scope, behavior, acceptance levels, and success criteria. |
| [03-Detailed Design](./specs/03-detailed-design.md) | Designing components, runtime ownership, failure semantics, and testing strategy. |
| [04-Development Tasks](./specs/04-development-tasks.md) | Decomposing tasks, dependencies, file ownership, interface responsibilities, and completion conditions. |
| [05-Document Review and Change Control](./specs/05-doc-review-change-control.md) | Managing approval, versioning, change propagation, candidate freeze, and evidence validity. |
| [06-Context Package](./specs/06-context-package.md) | Defining the generation, validation, exploration, invalidation, and reconciliation of Context Packages. |
| [07-Sub-Agent Delegation Prompts](./specs/07-subagent-delegation-prompts.md) | Defining role contracts, prompt structure, permissions, and output states. |
| [08-Verification, Review, and Merge Gates](./specs/08-verification-review-merge-gate.md) | Defining Levels 0–3, independent review, branch gates, and merge eligibility. |
| [09-Hermes Ledger Runtime](./specs/09-hermes-ledger-runtime.md) | Defining task ledgers, state consumption, pausing, resuming, scheduling handover (`CoordinatorEpoch`), carrier policy, and the dispatch gate. |
| [10-Terminology](./specs/10-terminology.md) | Establishing the repository-wide baseline for fixed terms, abbreviations, and official translations. |

Role prompt templates are located in [`agent-prompts`](./agent-prompts/README.md). Templates are the starting point for delegation; they cannot replace investigation of current facts, the task baseline, or an Agent’s autonomous exploration.

## Boundaries of Use

- Product objectives, business decisions, risk acceptance, and final authorization remain the responsibility of humans;
- Agents may not replace humans in performing unauthorized external operations;
- Operations rejected by security controls must be handed to a human for execution and may not be bypassed; privileged operations require authorization and an audit trail;
- Projects may use their own languages, frameworks, models, and branch strategies;
- Large projects should be decomposed by Sprint and delivered through multiple iterations;
- Test results must be interpreted together with design review, system verification, and acceptance in the real environment.

## Practical Background and Continuous Evolution

This standard comes from real AI-assisted and multi-Agent development practice. Problems discovered during development are organized through retrospectives into reusable processes, roles, evidence, and gates.

The standard will continue to evolve with practice. Each new rule should state the general risk it is intended to address, avoiding the direct codification of a one-time incident response as a universal requirement. Changes to the standard must also be reviewed, with their version, scope of impact, and propagation requirements made explicit.

## In Closing

Over more than ten years in the software industry, I have worked as a developer, architect, project lead, and department head. Throughout that journey, what has consistently made me happy is the process of starting from nothing, gradually turning an idea into a real product, and finally seeing it brought into existence.

AI is lowering the barrier to creating software, giving more people the opportunity to try things that once required a complete technical team. By sharing this standard, I hope to make the experience and methods I have accumulated over many years available to more people, so that even ordinary people without a technical background can use AI to start with an idea, create a product of their own, and experience the joy of going from zero to one.

## Current Version

- Standard version: V3.0 (finalized 2026-08-24)
- Standard status: Reviewed and approved (V2.5 is the previous baseline)
- Scope of application: New features, upgrades to existing features, defect fixes, technology migrations, data migrations, architectural refactoring, and other software development tasks
- Author and maintainer: Richy
- V3.0 revision: Tiffany-Dev (Hermes resident Coordinator); independent review: Codex
- Current scheduling implementation: Hermes (replaceable; the standard is not bound to it)

## V2.5 → V3.0 Upgrade Summary

V3.0 is not a feature list. It is an upgrade in **standard maturity**: each real problem exposed by the V2.5 production iteration has been converted into a general rule, a mechanical gate, or a data-driven policy.

### 1. Abstraction: concrete practices become portable rules

- **Contract and implementation separation**: the standard defines only role contracts, properties, and acceptance criteria; scripts, endpoints, and vendor configuration move to instance records. The standard is portable across projects and machines;
- **Execution carriers are no longer static**: available carriers and allocation rules are defined solely by the versioned **Carrier Policy Artifact**, with controlled and auditable changes;
- **Event sources are classified by function**: push, polling, write-back, human freeze, and scheduled reconciliation can change channels without changing the standard.

### 2. Process repair: closing gaps exposed by real incidents

Each item corresponds to a real incident from practice:

- **Scheduling-authority competition** (duplicate dispatch caused by dual drivers) → atomic conditional `CoordinatorEpoch` takeover plus authorized recovery after owner loss;
- **Record identity reuse** (two executions sharing one `RecordID`) → globally unique `RecordID` (ULID), conflict freeze, and mapping audit;
- **Verification-environment drift** (main checkout passes while clean checkout fails) → mandatory clean-checkout homomorphic gate for L2+ plus a declared configuration fallback source;
- **Verbal expansion of waivers** (different root causes hidden behind one error code) → seven-item waiver boundary tuple, invalid if incomplete;
- **“Completed but no conclusion”** (a response with no business result consumed as success) → execution-failure classification, one bounded resend, and an attempt limit preventing loops.

### 3. Mechanical gates: rules move from “read and remember” to “pass the gate”

- Fail-closed pre-dispatch gate: eight checks block dispatch, and a damaged policy rejects all dispatches;
- Every dispatch records `PolicyVersion` and the policy digest, making it possible to answer later why a carrier was selected.

### 4. Permission-model redesign

- Unified `danger-full-access` (the inability of the read-only sandbox to compile and verify was established by testing);
- **Code Immutability Constraint** offsets the permission expansion: Reviewers work in isolated verification workspaces and reconcile candidate integrity in both directions before and after review.

> The complete revision basis and ten-round review record are in `V3.0-proposal.md` on the `3.0` branch (archived historical proposal).

---

## Chinese Edition

The complete Chinese edition is maintained in a separate repository:

[AI Coding Standards](https://github.com/foreverdesmond/ai-coding-standards)
