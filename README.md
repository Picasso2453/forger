# Forger — Corporate Agent Workflow System

Documentation-first, disposable-agent workflow powered by Kilo Code.

---

## What is Forger?

Forger is a corporate-style AI agent workflow system that turns a project idea into running software through a structured, documentation-driven pipeline. A Planner agent conducts a discovery session with the user to define the project's scope, stack, and constraints. Once the user approves and locks the definition, a Project Manager agent breaks the work into milestones, bursts, and atomic slices — each backed by file contracts that define exactly what each output file must do, accept, and produce.

From there, an Orchestrator agent drives the execution loop. Disposable Developer agents implement one slice at a time from task packets — self-contained documents combining the slice definition and its contracts. After each slice, a QA agent validates the output. If validation fails, a Debugger agent is dispatched to fix the issue. When all slices are complete, an Auditor agent reviews the full codebase for compliance and writes a final audit report.

All workflow state lives in Markdown files. Agents have no memory between tasks — every agent reads the relevant docs, does its job, and exits. This makes the system deterministic, auditable, and fully reproducible.

---

## Prerequisites

- Python 3.11+
- Kilo Code VSCode extension
- An AI provider configured in Kilo Code (Claude, GPT-4, etc.)

---

## Quickstart

1. Clone or open this directory in VSCode.
2. Install the Kilo Code extension from the VSCode marketplace.
3. Run `python forger_cli.py init` in the terminal to create project structure and templates.
4. Open the Kilo Code panel and switch to the **forger-planner** mode.
5. Chat with the Planner to define your project — answer questions about stack, scope, target users, constraints, and definition of done.
6. When satisfied with the project definition, type: `locked`
7. The system automatically:
   - Locks `PROJECT.md` and sets `STATE.md` phase to PLANNING
   - Runs the PM agent to generate `MILESTONES.md`, `BURSTS.md`, `SLICES.md`, and file contracts
   - Runs the Orchestrator to create task packets in `tasks/`
   - Begins the DEV execution loop (slice by slice)
8. Watch tasks execute. Monitor progress at any time: `python forger_cli.py status`
9. When all slices are done, review `AUDIT.md` for the compliance report.

---

## Agent Modes

| Mode | Slug | Role | Triggered By |
|------|------|------|--------------|
| Planner/Architect | forger-planner | Discovery session with user, fills PROJECT.md | User (manually switch to this mode) |
| Project Manager | forger-pm | Generates MILESTONES, BURSTS, SLICES, contracts | Planner (after lock) |
| Orchestrator | forger-orchestrator | Manages execution loop, dispatches slices | PM (after planning) |
| Developer | forger-dev | Implements one slice from task packet | Orchestrator (per slice) |
| QA | forger-qa | Validates DEV output against contracts | Orchestrator (per slice) |
| Debugger | forger-debugger | Fixes QA failures | Orchestrator (on failure) |
| Auditor | forger-auditor | Full compliance check at project completion | Orchestrator (at completion) |

---

## Workflow Control

The system is entirely driven by agent mode switching in Kilo Code:

1. **Planner Mode** — Interactive discovery session with user → locks PROJECT.md
2. **PM Mode** (auto-spawned) — Generates MILESTONES.md, BURSTS.md, SLICES.md, and contracts
3. **Orchestrator Mode** (auto-spawned) — Begins wave-based execution loop
4. Monitor progress anytime: `python forger_cli.py status`

---

## CLI Reference

| Command | Description |
|---------|-------------|
| `init` | Create project structure and templates |
| `status` | Show phase, lock status, and slice progress |
| `lock` | Lock project scope (called by Planner after user says "locked") |
| `unlock` | Unlock project scope (re-enter discovery) |
| `plan` | Start planning phase (creates template files for PM) |
| `orchestrate` | Generate task packets from SLICES.md |
| `next-slice` | Print next pending slice ID and title (or ALL_DONE) |
| `start-slice ID` | Mark slice as in_progress |
| `complete-slice ID` | Mark slice as done |
| `fail-slice ID` | Mark slice as failed |
| `list-slices` | Show all slices with status icons |

Usage: `python forger_cli.py <command> [args]`

---

## Project File Structure

```
[project-dir]/
├── forger_cli.py                # CLI entry point (copied by forger init)
├── .kilocodemodes               # Kilo Code agent modes (copied by forger init)
├── [your-source-code]/          # Your project code (src/, tests/, etc.)
└── .forger/                     # Hidden directory — all workflow state
    ├── PROJECT.md               # Project definition (Planner writes here)
    ├── STATE.md                 # Workflow phase and lock status
    ├── MILESTONES.md            # Major milestones (PM generates)
    ├── BURSTS.md                # Burst breakdowns (PM generates)
    ├── SLICES.md                # Atomic task slices (PM generates)
    ├── AUDIT.md                 # Compliance audit report (Auditor writes)
    ├── prompts/                 # Role instruction files (copied by forger init)
    ├── contracts/               # File contracts (PM generates one per output file)
    ├── tasks/                   # Task packets (Orchestrator generates)
    │   ├── [S1].md              # Task packet for slice
    │   ├── [S1]-DONE.md         # DEV completion summary
    │   └── [S1]-DEBUG.md        # Debug summary (if QA failed)
    ├── qa_reports/              # QA validation reports
    │   └── [S1].md
    └── status/                  # Per-slice status files (concurrent execution support)
```

---

## Core Principles

1. All project state lives in Markdown files.
2. Documentation is the source of truth — not agent memory.
3. Agents are disposable and task-scoped — no state between tasks.
4. Separation of concerns is mandatory — each file has a contract.
5. Files define contracts agents must follow.
6. The system is deterministic — same inputs produce same outputs.

---

## Distribution & Installation

### Install Globally via pip

```bash
pip install forger-code
```

Then initialize your project:

```bash
# Recommended (works on all systems):
python -m forger_cli init

# Or if you added Python Scripts to PATH:
forger init
```

### Package Contents

When you install `forger-code`, you get:

```
forger_code/                    # Python package
├── __init__.py
└── data/
    ├── custom_modes.yaml       # 7 Kilo Code agent modes
    └── prompts/                # 8 role instruction files (108 KB)
        ├── PLANNER_ARCHITECT.md
        ├── PM.md
        ├── ORCHESTRATOR.md
        ├── DEV.md
        ├── QA.md
        ├── DEBUGGER.md
        ├── AUDITOR.md
        └── [others]

forger_cli.py                   # CLI entry point (installed as `forger` command)
```

### What `forger init` Does

1. Copies `forger_cli.py` to your project root
2. Creates `.kilocodemodes` from `custom_modes.yaml`
3. Creates `.forger/prompts/` with all role instruction files
4. Creates `.forger/PROJECT.md` template

---

## Troubleshooting

### `forger` command not found

**Recommended solution (works on all systems):**
```bash
python -m forger_cli init    # Instead of: forger init
python -m forger_cli status  # Instead of: forger status
```

**If you want to use the `forger` command directly**, add Python Scripts to your PATH:

**Windows PowerShell:**
```powershell
# Option 1: Add to current session only
$env:PATH += ";$([System.Environment]::GetEnvironmentVariable('APPDATA'))\Python\Python314\Scripts"

# Option 2: Add permanently (run as Administrator)
[Environment]::SetEnvironmentVariable(
  "PATH",
  [Environment]::GetEnvironmentVariable("PATH", "User") + ";$([System.Environment]::GetEnvironmentVariable('APPDATA'))\Python\Python314\Scripts",
  "User"
)
# Then restart PowerShell
```

**Linux/macOS:**
```bash
# Usually already on PATH, but if not:
export PATH="$HOME/.local/bin:$PATH"
```

### Package installation fails

If `pip install forger-code` fails, try:
```bash
pip install --upgrade pip setuptools wheel
pip install forger-code
```

---

## Known Limitations

- **Synchronous Execution Assumed**: The system assumes `new_task()` spawning is synchronous (task completes before parent continues). Async execution would require the Orchestrator to poll `SLICES.md` instead.
- **No Transient Failure Retry**: Infrastructure failures (Docker not running, port in use) are treated as hard failures. This is intentional — environment issues require human intervention.
- **Single Project Per Workspace**: Forger operates on the current working directory. Use separate directories for separate projects.
- **Model Selection**: Currently Kilo Code's global mode settings control which model is used. Adaptive slice-level model selection is designed but not yet implemented (see `ADAPTIVE_MODEL_TIERING.md`).

## Resources

- **ARCHITECTURE.md** — System design overview
- **AGENTS.md** — Agent role reference
- **IMPLEMENTATION_NOTES.md** — Implementation history and decisions
- **IMPROVEMENTS_TRACKER.md** — Known issues and improvements
- **docs/design/** — Future feature designs (Adaptive Model Tiering, Meta-Analyzer)
