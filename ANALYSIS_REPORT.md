# Forger Project Analysis Report

**Date:** 2026-03-05
**Mode:** Debug
**Analysis Scope:** forger_cli.py, prompt files, custom_modes.yaml, IMPLEMENTATION_NOTES.md

---

## Executive Summary

This analysis identified **32 potential issues** across the Forger codebase, categorized by severity. The analysis examined the CLI (`forger_cli.py`), 7 prompt files, and `custom_modes.yaml`. Many issues documented in `IMPROVEMENTS_TRACKER.md` (Round 12) remain unaddressed.

---

## Findings by Severity

### HIGH Severity Issues

| # | Issue | File | Description |
|---|---|---|---|
| H1 | Orchestrate overwrites packets | forger_cli.py | Running `orchestrate` twice during execution overwrites in-progress packets with no warning |
| H2 | Orchestrate missing locked check | forger_cli.py | Unlike `cmd_plan()`, orchestrate has no locked state validation |
| H3 | Duplicate Slice IDs undetected | forger_cli.py | `validate_slices()` doesn't check for duplicate IDs; second slice is permanently unreachable |
| H4 | STATE.md corruption silent | forger_cli.py | Missing `locked:` line returns "false" default; commands silently treat incomplete STATE as unlocked |
| H5 | QA cannot detect unauthorized modifications | QA.md | QA has no snapshot of original file state; cannot verify modifications outside contracts |
| H6 | Field regex too permissive | forger_cli.py | `parse_slices()` matches "Final Status" as valid field; `fields.get("status")` silently ignores it |

### MEDIUM Severity Issues

| # | Issue | File | Description |
|---|---|---|---|
| M1 | pytest assumption | QA.md | "no tests ran" detection uses pytest-specific output; unittest reports differently |
| M2 | Interrupt mode undocumented | ORCHESTRATOR.md | "Apply interrupt mode check" mentioned but no burst/milestone boundary comparison logic |
| M3 | Debugger winner ambiguous | ORCHESTRATOR.md | No specification for picking which Debugger's fix to apply if both succeed or neither does |
| M4 | STALE state not persisted | ORCHESTRATOR.md | "First vs second STALE" tracked in-memory only; crash between recoveries loses state |
| M5 | Malformed QA report unhandled | ORCHESTRATOR.md | Empty report or extra text in STATUS line not handled |
| M6 | Stub detection gaps | QA.md | SQLite `:memory:` or always-same UUID passes current anti-stub checks |
| M7 | Exception masking | forger_cli.py | `_find_forger_root()` catches all exceptions instead of just `ImportError` |
| M8 | DEV termination unspecified | DEV.md | After writing DONE.md, no instruction on HOW to terminate (attempt_completion vs. stopping) |
| M9 | PM trade-offs hidden | PM.md | Confirmation summary shows what was chosen, not what was rejected |
| M10 | Auditor missing logical checks | AUDITOR.md | 7 checks are structural only; no forward-flowing dependency or slice-to-milestone validation |

### LOW Severity Issues

| # | Issue | File | Description |
|---|---|---|---|
| L1 | PROJECT.md wording confusing | PLANNER_ARCHITECT.md | Step 1 says update now, but lock also touches PROJECT.md |
| L2 | Git/Obsidian preferences unused | PLANNER_ARCHITECT.md | Cluster 9 captures preferences but no subsequent agent reads them |
| L3 | Empty audit passes vacuously | AUDITOR.md | 0 slices = all checks pass trivially |
| L4 | in_progress not flagged | AUDITOR.md | pending/in_progress slices not flagged as incomplete |
| L5 | Debugger missing customInstructions | custom_modes.yaml | All other modes have customInstructions; Debugger relies on spawn message only |

---

## Root Cause Analysis

### 1. CLI State Validation Gaps

The CLI has multiple locations where state consistency is not validated:
- [`cmd_orchestrate()`](forger_cli.py:787) does not check locked state
- [`read_state()`](forger_cli.py:202) silently returns empty dict for corrupted STATE.md
- [`validate_slices()`](forger_cli.py:359) doesn't check for duplicate IDs

**Likely source:** Incremental feature additions without holistic validation design

### 2. Prompt Completeness Gaps

Several prompts contain references to functionality without implementation details:
- ORCHESTRATOR.md mentions "interrupt mode check" but provides no logic
- QA.md requires file modification detection without providing a mechanism
- AUDITOR.md has no logical dependency chain validation

**Likely source:** Design-by-intent without fully-specified implementation paths

### 3. State Persistence Issues

Runtime state that should be persistent is kept in-memory:
- STALE recovery attempts not persisted to disk
- "Second STALE -> fail" logic exists only in Orchestrator memory

**Likely source:** Assumptions about single-session execution that don't hold in practice

---

## Recommended Diagnosis Confirmation

Before fixing, the following should be confirmed:

1. **H1 (Orchestrate overwrite)**: Run `python forger_cli.py orchestrate` twice and verify second run overwrites without warning
2. **H2 (Missing locked check)**: Run `orchestrate` on unlocked project and verify it proceeds
3. **H3 (Duplicate IDs)**: Create SLICES.md with two `## S1` headings and verify no error
4. **H4 (STATE corruption)**: Delete `locked:` line from STATE.md and verify commands proceed as unlocked
5. **M4 (STALE persistence)**: Start a slice, kill session, restart and verify STALE detection works

---

## Logged Validations

To confirm these issues, add the following diagnostic logs:

```python
# In cmd_orchestrate() - validate locked state
if state.get("locked") != "true":
    print("ERROR: Project must be locked before orchestrate", file=sys.stderr)
    return

# In validate_slices() - check for duplicates
seen_ids = set()
for s in slices:
    if s["id"] in seen_ids:
        warnings.append(f"DUPLICATE: {s['id']}")
    seen_ids.add(s["id"])
```

---

## Comparison with IMPROVEMENTS_TRACKER.md

The issues identified align closely with IMPROVEMENTS_TRACKER.md Round 12 findings:

- **H1** = M3 in tracker (orchestrate overwrites)
- **H2** = M7 in tracker (no locked check)
- **H3** = M9 in tracker (duplicate IDs)
- **H4** = M10 in tracker (STATE corruption)
- **H5** = H1 in tracker (QA modifications)
- **H6** = M6 in tracker (field regex)
- **M4** = M4 in tracker (STALE persistence)
- **M10** = H2 in tracker (Auditor logic)

Most HIGH issues were previously identified but remain unfixed.

---

## Next Steps

1. **Confirm diagnosis** by running the validation steps above
2. **Prioritize fixes** - HIGH severity issues should be addressed first
3. **Implement incrementally** - Fix one category at a time
4. **Add tests** - After fixes, verify with E2E test project
