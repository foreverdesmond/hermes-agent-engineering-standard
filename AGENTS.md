# AGENTS.md — Context L1 Standard Entry Point

> Spec version: V3.0
> Document status: Approved (finalized 2026-08-24; final review by Richy)
> Author: Tiffany-Dev (V3.0 revision; initial version = WorkBuddy)
> Created: 2026-08-20
> Last updated: 2026-08-24
> Reviewer: Richy (approved)

---

## 1. Purpose

This file is the **Context L1 standard entry point** (the physical carrier) for this repository. It provides AI coding agents (the execution carriers designated by the current Carrier Policy Artifact) working in this repository with consistent, predictable, project-level operating instructions.

It answers one question: what cross-task, stable conventions, boundaries, and rules should an agent entering this repository know?

It connects to `specs/06-context-package.md`: `AGENTS.md` is the carrier of Context L1, and `06` defines the layering, triggering, expiry, and reconciliation rules for Context L1/L2/L3.

## 2. Layering and Priority (closest-wins)

`AGENTS.md` uses a layering + closest-wins mechanism:

```text
Global context entry point     # cross-repository conventions (host path mapped by the runtime)
  → ./AGENTS.md               # project-level: this repository's conventions (this file)
    → ./<subdir>/AGENTS.md    # subpackage-level: subdirectory-specific conventions (as needed)
```

Conflict-resolution priority (highest to lowest):

1. **The user's current explicit instruction** — highest, overrides everything;
2. **The closest `AGENTS.md`** — the `AGENTS.md` nearest the file being edited wins;
3. **Remote documents / parent `AGENTS.md`** — used as a fallback supplement.

An agent should read "the single `AGENTS.md` closest to the file it is currently working on," rather than loading all layers indiscriminately.

## 3. Project Overview

| Item | Description |
|---|---|
| Repository nature | **Multi-agent development collaboration standards library** (pure documentation repository, no source-code engineering) |
| Tech stack | Pure Markdown (`.md`); no build/compile/test commands |
| Spec status | V3.0-draft under revision (V2.5 is the previous approved baseline); proposal and body are on the `3.0` branch |
| Scheduling model | **Hermes resident Coordinator** + parallel execution by the carriers designated by the Carrier Policy Artifact |

## 4. Branch Strategy

| Branch | Role | Description |
|---|---|---|
| `main` | Stable branch | Approved baseline, merged by an independent merge task after authorization |
| `V2.5` (uppercase remote) | Current development branch | Hermes migration development branch |

**Branch naming warning**: historical V2.5 local and remote branch names differed in case (pushes require an explicit refspec; see the instance run record for the exact command); V3.0 revisions are made on the `3.0` branch.

## 5. Git and Tool Permissions (Sandbox Tiers)

Per `Hermes Capability Boundary List` §6, tool permissions are tiered by sandbox:

| Task type | sandbox | Description |
|---|---|---|
| All Codex dispatches (development/integration/documentation/Review/validation) | `danger-full-access` | V3.0 unified permission; Reviewers/Validators are subject to the Code Immutability Constraint (09 §7.3): they may build and test, but may not modify business source, candidate commits, or merge |

Rules:

- V2.5 read-only sandbox tests could not compile or run tests → from V3.0 onward, use unified `danger-full-access`, offsetting the permission expansion with the Code Immutability Constraint, isolated detached verification workspaces, and before/after HEAD/tree reconciliation;
- **The Coordinator (Hermes) does not perform git on their behalf**: worktree creation / commit / merge is handled by the respective role itself; Hermes only dispatches parameters, runs periodic reconciliation, and judges gates;
- Tasks with conflicts are not run in parallel; genuine conflicts during execution require Richy's coordination (see `Hermes Process & Boundary Resolution` C2);
- **Scheduling concurrency**: `CoordinatorEpoch` uniquely owns scheduling authority; cron and interactive sessions hand over ownership under 09 §12.3, and dual drivers are forbidden;
- **Dispatch gate**: every dispatch must pass the pre-dispatch gate (fail-closed) and record `PolicyVersion` + `PolicyArtifactDigest`.

## 6. Documentation Conventions

All specification documents in this repository follow unified conventions:

- **Document header**: spec version, document status, author, created date, last updated, reviewer, revision history (table at end);
- **Document status**: `Draft` / `Pending Review` / `In Revision` / `Approved` / `Pending Sync` / `Superseded` / `Archived`;
- **Versioning rules**: per `specs/05-doc-review-change-control.md` §5 (`Vmajor.minor`);
- **Terminology**: code baselines uniformly use `CodeBaseSHA` (the old name `BaseSHA` is deprecated);
- **Facts vs. inference separation**: verified facts, inferences, recommendations, and open items must be recorded separately.

## 7. Connection to This Repository's Specifications

| This file's role | Corresponding spec |
|---|---|
| Context L1 carrier | `specs/06-context-package.md` |
| Context-package generation responsibility & timing | `Hermes Process & Boundary Resolution` §III (Doc/Design Reviewer, generated before development once dependencies are satisfied) |
| Scheduling & ledger | `specs/09-hermes-ledger-runtime.md` |
| Roles & prompts | `specs/07-subagent-delegation-prompts.md` + `agent-prompts/` |

---

## Revision History

| Version | Date | Author | Change Description |
|---|---|---|---|
| V1.0 | 2026-08-20 | WorkBuddy | Initial creation: established Context L1 standard entry point, closest-wins priority, project overview, branch strategy, sandbox tiers, documentation conventions |
| V1.1 | 2026-08-20 | WorkBuddy | §3 spec status updated to V2.5 pending review (V2.3 is the previous approved baseline) |
| V2.5 final | 2026-08-20 | WorkBuddy | Reviewed and approved, marked as the official V2.5 baseline |

| V3.0-draft | 2026-08-24 | Hermes | Synchronized the V3.0 revision (proposal v5): updated §3 to V3.0-draft; changed §5 to unified danger-full-access plus the Reviewer Code Immutability Constraint; added scheduling concurrency (CoordinatorEpoch handover protocol) and pre-dispatch gate guidance. See `V3.0-proposal.md` on the `3.0` branch |

| V3.0 final | 2026-08-24 | Tiffany-Dev | Richy announced overall V3.0 approval: headers raised to V3.0/Approved; all ten review rounds (proposal v1-v5 plus nine body rounds) closed; D0/D1 residue-zero acceptance achieved; evidence pack E1-E8 and Canary 11/11 archived |
