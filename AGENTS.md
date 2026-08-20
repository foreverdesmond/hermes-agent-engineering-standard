# AGENTS.md — Context L1 Standard Entry Point

> Spec version: V2.5
> Document status: Approved (V2.5 final baseline)
> Author: WorkBuddy (delegated by the Coordinator—implemented by Hermes)
> Created: 2026-08-20
> Last updated: 2026-08-20
> Reviewer: Richy (approved)

---

## 1. Purpose

This file is the **Context L1 standard entry point** (the physical carrier) for this repository. It provides AI coding agents (WorkBuddy / Codex / others) working in this repository with consistent, predictable, project-level operating instructions.

It answers one question: what cross-task, stable conventions, boundaries, and rules should an agent entering this repository know?

It connects to `specs/06-context-package.md`: `AGENTS.md` is the carrier of Context L1, and `06` defines the layering, triggering, expiry, and reconciliation rules for Context L1/L2/L3.

## 2. Layering and Priority (closest-wins)

`AGENTS.md` uses a layering + closest-wins mechanism:

```text
~/.claude/AGENTS.md           # global: conventions across all repositories
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
| Spec status | V2.5 approved (V2.3 is the previous approved baseline); Hermes Coordinator migration in progress |
| Scheduling model | **Hermes resident Coordinator** + WorkBuddy / Codex / Human parallel collaborative execution |

## 4. Branch Strategy

| Branch | Role | Description |
|---|---|---|
| `main` | Stable branch | Approved baseline, merged by an independent merge task after authorization |
| `V2.5` (uppercase remote) | Current development branch | Hermes migration development branch |

**Case warning**: the local branch name is lowercase `v2.5`, the remote branch name is uppercase `V2.5`; local `v2.5` has no upstream tracking, so pushes must use an explicit refspec:

```text
git push origin v2.5:V2.5
```

## 5. Git and Tool Permissions (Sandbox Tiers)

Per `Hermes Capability Boundary List` §6, tool permissions are tiered by sandbox:

| Task type | sandbox | Description |
|---|---|---|
| Read-only research / code review | `read-only` | Reviewer, read-only investigation |
| Writing documents / writing evidence | `workspace-write` | Does not touch `.git` |
| **Requires git commit** | `danger-full-access` | `.git` is a protected path under `workspace-write` and will block git |

Rules:

- Roles that require `git add/commit/merge/push` (Implementer, Integrator) must use `danger-full-access`;
- **The Coordinator (Hermes) does not perform git on their behalf**: worktree creation / commit / merge is handled by the respective role itself; Hermes only dispatches parameters, runs periodic reconciliation, and judges gates;
- Tasks with conflicts are not run in parallel; genuine conflicts during execution require Richy's coordination (see `Hermes Process & Boundary Resolution` C2).

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
