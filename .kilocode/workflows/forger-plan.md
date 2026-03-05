# Forger Workflow: Planning Phase (forger-plan)

This workflow is executed by the `forger-pm` mode after the project has been locked by the Planner.
Follow every step in order. Do not skip steps. Do not proceed to the next step until the current step is complete.

---

## Step 1: Verify the Project Lock

Read `STATE.md` in the current working directory.

Check the value of `locked`. It must be `true`.

- If `locked: true` — proceed to Step 2.
- If `locked: false` or STATE.md does not exist — **stop immediately**. Tell the user:
  "The project is not locked. Please switch to forger-planner mode, complete the discovery session,
  and confirm the project by saying 'locked'. Do not proceed with planning until the project is locked."

---

## Step 2: Read the Project Definition

Read `PROJECT.md` in the current working directory in full.

Confirm it contains all of the following sections:
- Project Name
- Purpose
- Target Users
- Tech Stack
- Architecture
- UI/UX Expectations
- External Integrations
- Constraints and Non-Functional Requirements

If any section is missing or substantially empty, note the gap but continue — do not block on this.
Document any gaps in the planning summary at the end.

---

## Step 3: Analyze Project Scope

Before writing any documents, reason through the project scope internally:

- How complex is this project? (small / medium / large)
- What are the major independent domains of work? (e.g., data layer, API layer, UI layer, auth, integrations)
- What are the natural delivery boundaries? (what must exist before other things can be built?)
- What is the minimum viable delivery order?

Use this analysis to determine:
- Number of milestones (3-6, based on complexity and delivery order)
- Number of bursts per milestone (typically 2-5)
- Approximate number of slices per burst (1-5; prefer 2-3 for clarity)

Document your reasoning briefly before proceeding. This does not need to be written to a file.

---

## Step 4: Write MILESTONES.md

Create `MILESTONES.md` in the current working directory.

Use exactly this format for each milestone:

```
# Milestones

## M1 - [Milestone Name]
- **Goal:** [One clear sentence describing what this milestone achieves]
- **Deliverables:**
  - [Concrete output 1]
  - [Concrete output 2]
- **Success Criteria:**
  - [Measurable condition 1 — something that can be verified as true or false]
  - [Measurable condition 2]
- **Owner:** PM

## M2 - [Milestone Name]
...
```

Requirements:
- 3-6 milestones total.
- Milestones must be ordered by dependency (earlier milestones should not depend on later ones).
- Success criteria must be measurable and objective, not vague (e.g., "all unit tests pass" not "code is good").
- Every major area of the project must be covered by at least one milestone.

---

## Step 5: Write BURSTS.md

Create `BURSTS.md` in the current working directory.

Use exactly this format for each burst:

```
# Bursts

## M1.B1 - [Burst Name]
- **Goal:** [One sentence describing the burst's focused objective]
- **Slices:** S1, S2, S3

## M1.B2 - [Burst Name]
- **Goal:** [One sentence]
- **Slices:** S4, S5

## M2.B1 - [Burst Name]
...
```

Requirements:
- Every milestone must have at least one burst.
- Slice IDs must be globally unique and sequential (S1, S2, S3, ... across all bursts).
- A burst's slices must all be within the same milestone's scope.
- Bursts within a milestone should be ordered by dependency.

---

## Step 6: Write SLICES.md

Create `SLICES.md` in the current working directory.

Use exactly this format for each slice:

```
# Slices

## S1 - [Slice Name]
- **Burst:** M1.B1
- **Status:** pending
- **Description:** [Precise description of what the DEV agent must do. Be specific enough that the agent can complete this without asking any questions.]
- **Inputs:**
  - [File or data the agent must read, e.g., "src/models/user.py (does not exist yet — create it)"]
  - [Or: "contracts/src-models-user.contract.md"]
- **Outputs:**
  - [File path the agent must produce or modify]
- **Contracts:** contracts/filename1.contract.md, contracts/filename2.contract.md
- **Tests:** [Exact test command to run, e.g., "pytest tests/test_user.py -v", or "none"]

## S2 - [Slice Name]
...
```

Requirements for each slice:
- Must be atomic: one focused goal, completable in a single pass.
- Must not depend on outputs from a slice with a higher ID (no forward dependencies).
- Description must be specific enough for a DEV agent to act on without clarification.
- Inputs must list every file the DEV agent needs to read.
- Outputs must list every file the DEV agent will create or modify.
- Contracts must reference the contract file(s) governing each output file.
- Tests must be a runnable command or "none".
- Status must be `pending` for all slices at this stage.

---

## Step 7: Create File Contracts

For every file the project will produce (source files, test files, config files, scripts, etc.),
create a contract file at `contracts/[descriptive-name].contract.md`.

The descriptive name should reflect the file path with slashes replaced by hyphens, e.g.:
- `src/models/user.py` → `contracts/src-models-user.contract.md`
- `tests/test_api.py` → `contracts/tests-test-api.contract.md`

Create the `contracts/` directory if it does not exist.

Use exactly this format for each contract:

```
# Contract: [File Path]

- **File Path:** [relative path from project root, e.g., src/models/user.py]
- **Purpose:** [What this file does and why it exists in the project]
- **Inputs:** [Data, files, or external resources this file consumes at runtime or import time]
- **Outputs:** [What this file exposes, produces, or returns — functions, classes, data, side effects]
- **Dependencies:** [Other project files and external libraries this file imports or requires]
- **Allowed Operations:** [What the implementing DEV agent is permitted to do, e.g.:]
  - Create new file from scratch
  - Implement the following functions: [list them]
  - Do not modify existing functions in this file
- **Related Slices:** [Slice IDs that create or consume this file, e.g., S1, S4]
- **Success Criteria:**
  - [Verifiable condition 1, e.g., "File imports without error"]
  - [Verifiable condition 2, e.g., "Function X returns Y given input Z"]
  - [Verifiable condition 3, e.g., "All unit tests in tests/test_X.py pass"]
```

Requirements:
- Every output file referenced in SLICES.md must have a contract.
- Success criteria must be objectively verifiable by a QA agent reading the file.
- Allowed Operations must be specific enough to prevent scope creep.

---

## Step 8: Generate Task Packets

Run the following command in the current working directory:

```
python forger_cli.py orchestrate
```

This command reads SLICES.md and generates a task packet file in `tasks/` for each slice.
Each task packet (e.g., `tasks/S1.md`) bundles the slice definition and its contract content
into a self-contained instruction file for a DEV agent.

Wait for the command to complete. Verify that:
- The `tasks/` directory now exists.
- A `.md` file exists in `tasks/` for each slice defined in SLICES.md.

If the command fails, report the error to the user and do not proceed.

---

## Step 9: Print Planning Summary

Print a summary of everything generated in this planning session:

```
Planning Complete — Summary
===========================
Milestones:  [count] (M1 through Mx)
Bursts:      [count] (across all milestones)
Slices:      [count] (S1 through Sy, all status: pending)
Contracts:   [count] files in contracts/
Task Packets: [count] files in tasks/

Milestones:
  M1: [name]
  M2: [name]
  ...

Slices by Burst:
  M1.B1: S1, S2, S3
  M1.B2: S4, S5
  ...

Files under contract:
  [list all contract file paths]
```

---

## Step 10: Hand Off to Execution

Tell the user:

"Planning complete. All milestones, bursts, slices, contracts, and task packets have been generated.

To begin execution:
1. Switch to **forger-orchestrator** mode in Kilo Code.
2. Start a new task and say: **'Begin execution. Read SLICES.md and run the forger-execute workflow.'**

The Orchestrator will run the full execution loop: DEV → QA → (Debugger if needed) → complete or fail each slice, and finally audit the entire project."

Do not begin execution yourself. Your role as PM ends here.
