# AGENTS.md — Forger System Reference

This file is automatically loaded by Kilo Code into every agent's context.
Read this document before taking any action in this project.

---

## System Overview

Forger is a documentation-first, disposable-agent workflow system for building software projects with deterministic, reproducible processes. It models a corporate delivery structure — Planner/Architect, PM, Orchestrator, DEV, QA, Debugger, Auditor — where each role is embodied by a single-purpose AI agent.

**Markdown is the source of truth.** Every decision, plan, contract, implementation record, and audit finding lives in a Markdown file in this repository. Agents have no persistent memory. All state is read from and written to files. An agent that loses context can be recreated from files alone.

Forger runs inside Kilo Code (a VSCode extension that manages agent modes and `new_task()` calls). The CLI tool `forger_cli.py` handles state transitions and file scaffolding.

---

## Core Principles

- **All state in Markdown** — No database, no memory, no hidden state. If it is not in a file, it did not happen.
- **Documentation is truth, not memory** — Agents are disposable. Files outlive them. Every agent reads files; no agent relies on conversation history.
- **Separation of concerns is mandatory** — Each file is owned by one role. Each agent operates within its defined scope. Crossing ownership boundaries is a violation.
- **Agents are disposable and task-scoped** — Every agent receives a task packet, does its job, writes its output to files, and terminates. No agent accumulates cross-session state.
- **Files define contracts agents must follow** — The `contracts/` directory contains per-file specifications. DEV agents implement to contracts. QA agents validate against contracts. No implementation happens without a contract.

---

## How to Start a New Project

1. Open Kilo Code in VSCode.
2. Switch to **forger-planner** mode (the Planner/Architect agent).
3. Chat naturally to describe what you want to build. The Planner/Architect will ask clarifying questions about scope, stack, architecture, users, constraints, and integrations.
4. When you are satisfied with the project definition, say **"locked"**.
5. The Planner/Architect will:
   - Run `python forger_cli.py lock` to freeze PROJECT.md and STATE.md.
   - Spawn the PM agent via `new_task()` to begin the planning pipeline.
6. The pipeline continues automatically: PM → Orchestrator → DEV/QA loops → Auditor.

---

## Agent Roles

| Role | File | Activated When | Responsibility |
|---|---|---|---|
| **Planner/Architect** | `prompts/PLANNER_ARCHITECT.md` | Project start (DISCOVERY phase) | Runs discovery conversation, produces PROJECT.md, locks scope |
| **PM** | `prompts/PM.md` | After lock (PLANNING phase) | Reads PROJECT.md, writes MILESTONES.md, BURSTS.md, SLICES.md, contracts/ |
| **Orchestrator** | `prompts/ORCHESTRATOR.md` | After planning (EXECUTING phase) | Runs execution loop, spawns DEV and QA agents per slice, tracks progress |
| **DEV** | `prompts/DEV.md` | Per slice, spawned by Orchestrator | Implements one slice by reading its task packet and contracts |
| **QA** | `prompts/QA.md` | Per slice, spawned by Orchestrator after DEV | Validates DEV output against contracts and success criteria |
| **Debugger** | `prompts/DEBUGGER.md` | Per slice failure, spawned by Orchestrator | Diagnoses QA failures, proposes and applies minimal fixes |
| **Auditor** | `prompts/AUDITOR.md` | After all slices complete (AUDITING phase) | Performs final compliance check across all artifacts, writes AUDIT.md |

---

## Key Files Reference

| File | Owner | Description |
|---|---|---|
| `PROJECT.md` | Planner/Architect | Project definition: identity, scope, stack, architecture, constraints, assumptions, DoD. Frozen after lock. |
| `STATE.md` | CLI (`forger_cli.py`) | Machine-readable workflow state: phase, locked flag, locked_at timestamp, last_updated. Frozen after lock. |
| `MILESTONES.md` | PM | High-level delivery milestones (3–6). Each has a goal, deliverables, and success criteria. |
| `BURSTS.md` | PM | Milestones broken into focused work bursts (2–5 slices each). |
| `SLICES.md` | PM (created), Orchestrator/CLI (status updates) | Atomic implementation units. Each slice has a Status field (pending / in_progress / done / failed). |
| `contracts/*.md` | PM | Per-file specifications. One contract per output file. All 8 required fields must be present. |
| `tasks/*.md` | Orchestrator | Per-slice task packets assembled by `forger_cli.py orchestrate`. Read by DEV agents. |
| `tasks/[id]-DONE.md` | DEV | Completion summary written by DEV. Lists files created/modified, test results, issues encountered. |
| `tasks/[id]-DEBUG.md` | Debugger | Debug report: root cause, fix options, chosen fix, changes made. |
| `qa_reports/*.md` | QA | QA evaluation report. Contains STATUS: PASS or STATUS: FAIL, itemized criteria checklist, summary. |
| `AUDIT.md` | Auditor | Final compliance report. Overall PASS/FAIL verdict, itemized findings for every audit check. |

---

## Workflow States

```
DISCOVERY  -->  (user says "locked")  -->  PLANNING  -->  (PM runs orchestrate)  -->  EXECUTING
                                                                                          |
                                                                             (all slices done/failed)
                                                                                          |
                                                                                       AUDITING  -->  COMPLETE
```

State transitions are performed by `forger_cli.py` commands:
- `lock` transitions DISCOVERY → PLANNING
- `orchestrate` transitions PLANNING → EXECUTING
- Auditor writes AUDIT.md while in AUDITING phase

STATE.md always reflects the current phase. Read it first if you are unsure where the project stands.

---

## CLI Quick Reference

All commands are run as `python forger_cli.py <command>` from the project root.

| Command | Description |
|---|---|
| `init` | Create project structure: scaffold STATE.md, PROJECT.md, MILESTONES.md, BURSTS.md, SLICES.md, and all directories |
| `status` | Show current phase, lock status, locked_at timestamp, and slice progress summary |
| `lock` | Lock project scope — sets `locked: true` and `locked_at` timestamp in STATE.md and PROJECT.md |
| `unlock` | Unlock project scope (emergency use only; invalidates planning docs) |
| `plan` | Start PLANNING phase — transitions STATE.md to PLANNING, ensures planning doc templates exist |
| `orchestrate` | Generate task packets from SLICES.md into `tasks/[id].md`, transition STATE.md to EXECUTING |
| `next-slice` | Print the next slice with status=pending, or print ALL_DONE if none remain |
| `start-slice [id]` | Mark a slice as in_progress (e.g., `start-slice S3`) |
| `complete-slice [id]` | Mark a slice as done and update STATE.md last_updated |
| `fail-slice [id]` | Mark a slice as failed |
| `list-slices` | Print all slices with status icons: `[ ]` pending, `[~]` in_progress, `[x]` done, `[!]` failed |

---

## Locked File Rules

After `forger_cli.py lock` is run, the following files are **frozen**:

| File | Frozen After | Who May Edit |
|---|---|---|
| `PROJECT.md` | Lock | Nobody (read-only for all agents) |
| `STATE.md` | Lock | CLI only (`forger_cli.py` commands) |
| `MILESTONES.md` | PM completes planning | Nobody |
| `BURSTS.md` | PM completes planning | Nobody |
| `SLICES.md` | PM completes planning (Status field excepted) | CLI only (status updates via complete-slice / fail-slice) |
| `contracts/*.md` | PM completes planning | Nobody |

Files that agents **may always write to** (within their role scope):
- `tasks/[id]-DONE.md` — DEV only
- `tasks/[id]-DEBUG.md` — Debugger only
- `qa_reports/[id].md` — QA only
- `AUDIT.md` — Auditor only

---

## Slice Status Values

| Status | Meaning |
|---|---|
| `pending` | Not yet started. Orchestrator will assign this slice to a DEV agent next. |
| `in_progress` | DEV agent is currently implementing this slice. Set by `start-slice`. |
| `done` | DEV completed implementation and QA passed. Set by `complete-slice`. |
| `failed` | DEV completed but QA failed after max retries (2). Set by `fail-slice`. Pipeline continues. |

A slice moves through the lifecycle: `pending` → `in_progress` → `done` (or `failed`).
A `failed` slice is documented and skipped — the pipeline does not halt on failure.

---

## Contract Required Fields

Every file in `contracts/` must contain all 8 of these sections:

| Field | Description |
|---|---|
| **File Path** | Relative path to the output file this contract governs (e.g., `src/api/users.py`) |
| **Purpose** | What this file does and why it exists in the system |
| **Inputs** | Data, parameters, files, or environment values this module receives |
| **Outputs** | Data, files, return values, or side effects this module produces |
| **Dependencies** | Other files, modules, packages, or external services this file depends on |
| **Allowed Operations** | Which agents may create or edit this file (e.g., `create: DEV`, `edit: DEV`) |
| **Related Slices** | Slice IDs this contract is associated with (e.g., `S2, S4`) |
| **Success Criteria** | Specific, testable conditions that confirm this file is correctly implemented |

A contract with any missing field is considered incomplete and will cause an Auditor FAIL.
