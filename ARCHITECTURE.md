# ARCHITECTURE - Forger

## System Overview
Forger is a documentation-first workflow that models corporate delivery roles (Planner/Architect, PM, DEV, QA, Auditor, Debugger, Orchestrator). Each role produces or validates Markdown artifacts that become the source of truth for subsequent steps. Agents are disposable and task-scoped.

## Core Artifacts
- `PROJECT.md`: Project definition (Planner/Architect)
- `MILESTONES.md`: High-level milestones (PM)
- `BURSTS.md`: Milestone broken into bursts (PM)
- `SLICES.md`: Actionable slices per burst (PM)
- `contracts/*.md`: File contracts (PM)
- `tasks/*.md`: Slice task packets (Orchestrator)

## Execution Flow
1. Discovery: Planner/Architect writes `PROJECT.md`.
2. Lock: User replies `LOCKED`; project is frozen.
3. Plan: PM generates milestones, bursts, slices, and file contracts.
4. Orchestrate: Orchestrator emits per-slice task packets.
5. Execute: DEV agents implement slices. QA validates. Auditor checks compliance. Debugger resolves failures.

## Separation of Concerns
Each file is owned by a specific role and has a contract defining purpose, inputs, outputs, dependencies, and success criteria. No agent reads beyond its task packet and relevant contracts.

## Determinism
All project state is stored in Markdown files to minimize context requirements and ensure reproducibility. Agents do not retain memory beyond their task.

