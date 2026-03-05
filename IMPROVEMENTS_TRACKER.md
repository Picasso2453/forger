# Forger Improvements Tracker

All improvement proposals evaluated across analysis rounds. Updated as items are implemented or re-evaluated.

## Legend

| Status | Meaning |
|--------|---------|
| DONE | Implemented |
| YES | Accepted, pending implementation |
| FUTURE | Deferred — good idea, not now |
| NO | Rejected — reason documented |

---

## Implemented

| # | Improvement | Round | Notes |
|---|---|---|---|
| 1 | `next-slice` STALE/QA_PENDING priority fix | R10 | STALE/QA_PENDING checked before pending; was returning pending S8 while S1 was stuck |
| 2 | `QA_PENDING:` prefix | R10 | Handles session death between DEV-done and QA-not-run |
| 3 | STALE retry limit (2nd STALE → fail-slice) | R10 | Prevented infinite STALE recovery loop |
| 4 | `[Contract not found]` guard in DEV.md | R10 | DEV writes BLOCKED DONE.md and terminates instead of attempting implementation |
| 5 | Parser `\s*` instead of `\s+` | R10 | Tolerates `-Field:` (no space after hyphen) PM formatting error |
| 6 | PM contract count quality gate (Step 7) | R10 | PM must verify contract count matches unique contract paths in SLICES.md |
| 7 | `cmd_orchestrate()` missing contract warnings | R10 | Reports missing contracts per slice after packet generation |
| 8 | `cmd_init()` old `forger/` directory warning | R10 | Prevents dual-directory agent confusion |
| 9 | `validate_slices()` function | R10 | Checks required fields, valid status values, contract presence; called in status + orchestrate |
| 10 | Prompt centralization (delete root `prompts/`) | R10 | `forger_code/data/prompts/` is single source; `_find_forger_root()` resolves to it |
| 11 | Success criteria quality rules in PM.md | R11 | Each criterion must contain action verb + specific outcome; vague words forbidden |
| 12 | `forger reset` command | R11 | Resets to PLANNING phase; wipes tasks/qa_reports/SLICES.md/MILESTONES.md/BURSTS.md; keeps PROJECT.md + contracts |
| 13 | Better `list-slices` output (burst grouping) | R11 | Groups slices by burst, shows per-burst and overall progress counters |
| 14 | Wave-based execution model (`next-wave`) | R13 | Returns all currently unblocked pending slices; DAG computed from Inputs/Outputs; enables parallel execution when async new_task() available |
| 15 | DAG cycle detection in `cmd_orchestrate()` | R13 | Validates dependency graph is acyclic before generating packets; handles incremental update pattern (multi-producer files) correctly |
| 16 | Per-slice sidecar status files | R13 | `.forger/status/[id]` atomic per-slice writes; `load_slices()` applies sidecar overrides; eliminates SLICES.md write contention for concurrent agents |
| 17 | PM composability design guidance | R13 | Step 4 wave design rules: explicit I/O, consistent file names, no URLs in Inputs, separate independent outputs, no shared state across slices |
| 18 | Fail-save: user pause on max retries and BLOCKED | R13 | After 3rd QA failure: writes FAILED.md + `ask_followup_question` continue/stop; BLOCKED state also pauses instead of silently halting |

---

## Deferred (Future)

| # | Improvement | Notes |
|---|---|---|
| 14 | MCP Server integration | Removes `execute_command` fragility; replace stdout parsing with typed tool calls; highest long-term impact |
| 15 | Git integration (auto-commit per slice) | Useful audit trail; needs git repo assumption; deferred until core is stable |
| 16 | `FORGER_HOME` env variable | Low priority; `_find_forger_root()` already handles pip + source layout |
| 17 | Template library (pre-built PROJECT.md starters) | Nice-to-have; deferred until core is stable |

---

## Rejected

| # | Improvement | Reason |
|---|---|---|
| 18 | Role-based contract views (DEV sees 3 of 8 fields) | DEV needs Dependencies for imports, Purpose for context; stripping causes wrong implementations |
| 19 | Contract versioning (SHA hash) | Over-engineering; mid-execution contract updates not a real problem in practice |
| 20 | Stub detection automation (CLI patterns) | Language-specific; QA.md already has explicit anti-stub checks |
| 21 | "Dumb DEV" principle | Same as #18; context stripping is the wrong direction |
| 22 | CDD / YAML contracts | Big architectural shift; current markdown is agent-friendly and human-readable |
| 23 | Slice chaining / function-level dependencies | Premature; requires parallel execution model that doesn't exist yet |
| 24 | Contract signing by DEV (implementation notes in contract) | DONE.md already serves this purpose |
| 25 | Live self-test commands in contracts | Redundant with Tests field; adds embedded-code complexity |
| 26 | Snapshot-based validation | Over-engineering; no evidence unintended file modification is a real problem |
| 27 | Token budget per slice | Not implementable; DEV agents cannot track their own token consumption |
| 28 | Modular CLI package (split forger_cli.py into modules) | 789 lines is manageable; splitting adds import complexity for no functional gain |
| 29 | Parallel slice execution | Needs fundamentally different state model; sequential execution is intentional |
| 30 | Web dashboard | Overkill for a markdown-first system |
| 31 | Multi-project management | Overkill; single-project focus is intentional |
| 32 | Slice dependency graph (DAG execution) | Premature; sequential execution covers all current cases |

---

## Round 12 Analysis Findings (Pending Decision)

Full deep-dive analysis across all 7 prompts, CLI, and custom_modes.yaml. Items below are candidates — not yet accepted or rejected.

### HIGH Severity

| # | Issue | File | Description |
|---|---|---|---|
| H1 | QA "unauthorized file modifications" check is unimplementable | QA.md | QA has no snapshot of original file state; cannot detect modifications outside contracts |
| H2 | Auditor missing logical consistency checks | AUDITOR.md | 7 checks are structural only; no check for forward-flowing slice dependencies or slice-to-milestone correspondence |
| H3 | `cmd_orchestrate()` silently overwrites existing task packets | forger_cli.py | Running orchestrate twice during execution overwrites in-progress packets with no warning |

### MEDIUM Severity

| # | Issue | File | Description |
|---|---|---|---|
| M1 | QA test framework assumed to be pytest | QA.md | "no tests ran" detection uses pytest-specific output; `unittest` reports "Ran 0 tests" which may be missed |
| M2 | Interrupt mode check logic undocumented | ORCHESTRATOR.md | "Apply interrupt mode check" is mentioned but zero implementation detail — no burst/milestone boundary comparison logic specified |
| M3 | Parallel Debugger winner selection ambiguous | ORCHESTRATOR.md | No specification for how to pick which Debugger's fix to apply if both succeed, or if neither succeeds |
| M4 | STALE recovery state not persisted | ORCHESTRATOR.md | "First vs second STALE" tracked in-memory only; Orchestrator crash after first recovery treats next session as first STALE again |
| M5 | QA report malformed file not handled | ORCHESTRATOR.md | Report exists but empty, or STATUS line has extra text (e.g. `STATUS: PASS (3/3)`), is not handled |
| M6 | Field name regex too permissive | forger_cli.py | `parse_slices()` matches "Final Status" as a valid field name; `fields.get("status")` then silently ignores it |
| M7 | `cmd_orchestrate()` doesn't validate locked state | forger_cli.py | Unlike `cmd_plan()`, orchestrate has no locked check; runs against any SLICES.md it finds |
| M8 | Stub detection heuristics miss sophisticated cases | QA.md | SQLite `:memory:` or always-same UUID passes all current anti-stub checks |
| M9 | Duplicate Slice IDs not detected | forger_cli.py | `validate_slices()` doesn't check for two slices with same ID; second is permanently unreachable |
| M10 | Corrupted STATE.md fails silently | forger_cli.py | Missing `locked:` line returns "false" default; all commands silently treat incomplete STATE as unlocked |

### LOW Severity

| # | Issue | File | Description |
|---|---|---|---|
| L1 | "Last moment to touch PROJECT.md" wording confusing | PLANNER_ARCHITECT.md | Step 1 says update now (before lock), but `forger lock` also touches PROJECT.md in Step 3; technically fine but misleading |
| L2 | Git/Obsidian captured in Planner but never acted on | PLANNER_ARCHITECT.md | Cluster 9 asks about git/Obsidian preferences; no subsequent agent reads or acts on these fields |
| L3 | Empty project passes audit vacuously | AUDITOR.md | 0 slices means all 7 checks pass trivially; no guard for "no work done" scenario |
| L4 | in_progress/pending slices not flagged by Auditor | AUDITOR.md | Check 7 requires DONE.md for done/failed slices, but pending/in_progress slices (incomplete execution) are not flagged as failures |
| L5 | `_find_forger_root()` catches all exceptions | forger_cli.py | `except Exception: pass` masks real errors (disk errors, permission errors); should catch only `ImportError` |
| L6 | DEV termination method not specified | DEV.md | After writing DONE.md, DEV should terminate but no instruction on HOW (attempt_completion vs. just stopping) |
| L7 | Debugger has no `customInstructions` block | custom_modes.yaml | All other modes have customInstructions; Debugger relies entirely on the spawn message + reading DEBUGGER.md |
| L8 | PM rejected option trade-offs not shown in confirmation summary | PM.md | Step 6 says PM considers 1-3 options and picks one; confirmation summary (Step 8) only shows what was chosen, not what was rejected |

---

## Round 13 — Debug Mode Analysis (2026-03-05)

Deep-dive analysis using debug methodology. Findings align closely with Round 12 - most issues were previously identified but remain unfixed.

### New Findings (Not in Round 12)

| Severity | Issue | File | Description |
|----------|-------|------|-------------|
| HIGH | Orchestrate overwrites in-progress packets | forger_cli.py | Running orchestrate twice during execution silently overwrites task packets |
| HIGH | Duplicate Slice IDs undetected | forger_cli.py | validate_slices() doesn't check for two slices with same ID |
| HIGH | Field regex too permissive | forger_cli.py | parse_slices() matches "Final Status" as valid field name |
| MEDIUM | Interrupt mode boundary logic missing | ORCHESTRATOR.md | No implementation of burst/milestone comparison |
| MEDIUM | Debugger winner selection ambiguous | ORCHESTRATOR.md | No spec for picking winner if both Debuggers succeed/fail |
| LOW | Git/Obsidian captured but unused | PLANNER_ARCHITECT.md | Preferences captured in Cluster 9 but no agent reads them |

### Validation Recommendations

Before fixing, confirm these issues with tests:
1. Run `orchestrate` twice - verify second run overwrites without warning
2. Create SLICES.md with duplicate IDs - verify no error
3. Add "Final Status" field - verify it matches incorrectly

---

## Notes

- Rounds 1–9: See `IMPLEMENTATION_NOTES.md` for detailed bug/fix history
- Round 10: mt5-api-docker run 1 — exposed priority bug, missing contracts, STALE loop
- Round 11: Architecture analysis (2 reports) — most proposals rejected; quality rules + 2 CLI improvements kept
- Round 12: Deep analysis across all prompts + CLI — 3 HIGH, 10 MEDIUM, 8 LOW findings logged above (pending decision)
- Round 13: Debug mode analysis (this report) — 6 HIGH, 10 MEDIUM, 5 LOW new findings; validates Round 12 findings
- MCP integration (#14) is the highest-impact deferred item — revisit when Kilo Code MCP tooling matures
