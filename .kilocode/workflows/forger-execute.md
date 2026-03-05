# Forger Workflow: Execution Loop (forger-execute)

This workflow is executed by the `forger-orchestrator` mode after the planning phase is complete.
The Orchestrator coordinates all DEV, QA, and Debugger agents but never implements code itself.

Follow every step precisely. Track retry counts per slice. Do not skip the audit phase.

---

## Prerequisites

Before starting the execution loop, verify:

1. `SLICES.md` exists and contains at least one slice with `Status: pending`.
2. `tasks/` directory exists and contains a `.md` task packet for each slice.
3. `STATE.md` exists and shows `locked: true`.

If any prerequisite is missing, stop and tell the user what is missing before proceeding.

---

## Execution Loop

Repeat the following loop until all slices are in a terminal state (done or failed).

Maintain a per-slice retry counter (starts at 0). Reset it when moving to a new slice.

---

### Step 1: Find the Next Pending Slice

Run:
```
python forger_cli.py next-slice
```

Interpret the output:
- If the output is `ALL_DONE` — all slices are in a terminal state. Exit the loop and go to **Step 8: Final Audit**.
- Otherwise — the output is a slice ID (e.g., `S3`). Note this ID. Proceed to Step 2.

---

### Step 2: Mark Slice as In Progress

Run:
```
python forger_cli.py start-slice [id]
```

Replace `[id]` with the slice ID from Step 1 (e.g., `S3`).

This updates STATE.md to mark the slice as `in_progress`.

Confirm the command succeeded before continuing.

---

### Step 3: Spawn DEV Agent

Spawn a new task in `forger-dev` mode with the following message:

```
You are a Forger DEV agent assigned to slice [id].

Read your task packet at tasks/[id].md.
Read each contract file listed in the Contracts section of the packet.
Implement every file listed in the Outputs section, satisfying all success criteria in the contracts.
Follow the Allowed Operations defined in each contract — do not create files not listed.
Run the test command specified in the Tests section. Record the results.
Write a completion summary to tasks/[id]-DONE.md.
Then run: python forger_cli.py complete-slice [id]
```

Wait for the DEV agent to complete before proceeding to Step 4.

If the DEV agent fails to complete or reports a critical error, log the failure to `tasks/[id]-ERROR.md`
and go to Step 7 (treat as QA FAIL with retry count already at maximum to force fail-slice).

---

### Step 4: Spawn QA Agent

After DEV completes, spawn a new task in `forger-qa` mode with the following message:

```
You are a Forger QA agent assigned to slice [id].

Read tasks/[id].md for the full slice context.
Read each contract file listed in the Contracts section.
Read the implementation files listed in the Outputs section.
For each success criterion in each contract, verify it explicitly against the implementation.
Write your QA report to qa_reports/[id].md with a PASS or FAIL verdict and a full checklist.
Do not modify any implementation files.
```

Wait for the QA agent to complete before proceeding to Step 5.

Ensure the `qa_reports/` directory exists before spawning QA. If it does not exist, create it.

---

### Step 5: Read QA Report and Route

Read `qa_reports/[id].md`.

Find the line: `**Verdict:** PASS` or `**Verdict:** FAIL`

**If PASS:**
- Run: `python forger_cli.py complete-slice [id]`
- Log: "Slice [id] completed successfully."
- Reset the retry counter for this slice.
- Return to **Step 1**.

**If FAIL:**
- Check the retry counter for this slice.
- If retry count is 0 (first failure): increment retry count to 1 and go to **Step 6: Spawn Debugger**.
- If retry count is 1 (second failure): go to **Step 7: Fail the Slice**.

---

### Step 6: Spawn Debugger Agent (First Failure Only)

Spawn a new task in `forger-debugger` mode with the following message:

```
You are a Forger Debugger agent assigned to slice [id].

Read qa_reports/[id].md to understand exactly what criteria failed.
Read tasks/[id].md for full slice context.
Read the implementation files that failed QA.
Propose 1-3 specific fix options with tradeoffs and confidence levels.
Choose the best option and apply the fix.
Write a debug summary to tasks/[id]-DEBUG.md including: QA failures addressed, options considered,
chosen fix with rationale, files modified, and any residual concerns.
Do not run QA yourself. Do not mark the slice complete.
```

Wait for the Debugger to complete.

After the Debugger finishes, return to **Step 3** (re-run DEV to re-implement with the fix in place,
then QA will re-validate). The retry counter is now 1 — if QA fails again, the slice will be failed.

---

### Step 7: Fail the Slice (Second Failure)

The slice has failed QA twice. Do not retry further.

Run:
```
python forger_cli.py fail-slice [id]
```

Log the failure:
- Write `tasks/[id]-FAILED.md` with:
  - Slice ID
  - Number of attempts made (2)
  - Summary of QA failures from `qa_reports/[id].md`
  - Reference to debug summary in `tasks/[id]-DEBUG.md` (if it exists)
  - Recommendation for human review

Print a notification:
"Slice [id] has been marked as FAILED after 2 QA attempts. Human review required.
See tasks/[id]-FAILED.md for details. Continuing to next slice."

Reset the retry counter and return to **Step 1**.

---

### Step 8: Final Audit

The execution loop has completed. All slices are in a terminal state (done or failed).

Before spawning the Auditor, collect a brief execution summary:

- Count of slices with status `done`
- Count of slices with status `failed`
- List any failed slice IDs

Spawn a new task in `forger-auditor` mode with the following message:

```
You are the Forger Auditor. The execution loop has completed.

Execution summary:
- Slices completed: [count]
- Slices failed: [count]
- Failed slice IDs: [list or "none"]

Read all project documentation:
- STATE.md, PROJECT.md
- MILESTONES.md, BURSTS.md, SLICES.md
- All files in contracts/
- All files in tasks/
- All files in qa_reports/

Produce AUDIT.md with a comprehensive compliance audit and an overall PASS or FAIL verdict.
Check: lock compliance, slice coverage, contract completeness, QA coverage, QA results, and failed slice documentation.
```

Wait for the Auditor to complete.

---

### Step 9: Print Final Execution Report

After the Auditor writes `AUDIT.md`, read it and print a final report to the user:

```
Forger Execution Complete
=========================
Project: [project name from PROJECT.md]
Date: [today's date]

Execution Results:
  Slices completed (PASS): [count]
  Slices failed:           [count]
  Total slices:            [count]

Audit Verdict: [PASS or FAIL from AUDIT.md]

[If FAIL: list the audit findings that caused the FAIL verdict]

Files produced:
  [List all non-planning files created during execution, grouped by directory]

Next steps:
  - Review AUDIT.md for the full compliance report.
  [If any slices failed:] - Review tasks/[id]-FAILED.md files for failed slices requiring human intervention.
  [If audit PASS:] - The project is complete and has passed all quality checks.
```

---

## Error Handling Reference

| Situation | Action |
|-----------|--------|
| `next-slice` returns `ALL_DONE` immediately (no slices pending) | Go directly to Step 8 (Final Audit) |
| DEV agent fails to complete | Log to `tasks/[id]-ERROR.md`, treat as second failure, go to Step 7 |
| QA agent fails to write report | Re-spawn QA once; if still no report, treat as FAIL and increment retry |
| `forger_cli.py` command fails | Report error to user, do not proceed until resolved |
| Debugger fails to complete | Log the failure, proceed to Step 7 anyway (second failure path) |
| `qa_reports/` directory missing | Create it before spawning QA agent |
| `tasks/[id].md` not found | Check orchestrate was run; if missing, re-run `python forger_cli.py orchestrate` |

---

## State Transition Reference

Each slice moves through these states (managed by `forger_cli.py`):

```
pending → in_progress → done
                     → failed
```

- `pending`: Initial state set by PM during planning.
- `in_progress`: Set by `start-slice [id]`.
- `done`: Set by `complete-slice [id]` after QA PASS.
- `failed`: Set by `fail-slice [id]` after 2 QA failures.

Never manually edit STATE.md. Always use the CLI commands to transition state.
