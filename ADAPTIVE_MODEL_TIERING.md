# Adaptive Model Tiering for Forger

**Status:** Design / discussion phase. Not yet implemented.
**Created:** 2026-03-05

---

## Problem

Every Forger agent currently uses the same model (user's Kilo Code default). Simple slices waste money if run on expensive models. Hard slices fail when run on free models — but blindly escalating through every tier wastes money on failed attempts.

The goal: **pick the right model on the first paid attempt**, rather than climbing a ladder blindly.

---

## Core Insight

Two signals are available to inform model selection — both free:

1. **Task packet content** (available at `orchestrate` time) — technology keywords, output count, test type
2. **QA failure reason** (available after first failure) — environment vs. code vs. specialized knowledge gap

---

## Architecture

### Signal 1: Pre-Classification (Deterministic, Zero Cost)

`forger_cli.py` runs `classify_slice_complexity()` during `orchestrate` and stamps a `Complexity-Tier` field into each task packet header.

```python
def classify_slice_complexity(slice_data: dict) -> str:
    text = " ".join([description, outputs, tests, contracts_text]).lower()

    # Tier 2: requires specialized or environment-specific knowledge
    specialized = ["wine", "mql5", "mt5", "dll", "winsock", "firmware",
                   "kernel", "tls", "ssl", "async def.*websocket", "cuda"]
    if any(kw in text for kw in specialized):
        return "specialized"

    # Tier 1: integration/infrastructure tasks
    complex_kw = ["docker", "database", "postgresql", "redis", "migration",
                  "authentication", "oauth", "websocket", "nginx", "systemctl"]
    num_output_files = len([f for f in outputs.split(",") if f.strip()])
    if any(kw in text for kw in complex_kw) or num_output_files >= 3:
        return "complex"

    return "standard"
```

Task packet header gets: `Complexity-Tier: standard | complex | specialized`

### Signal 2: Failure Reason Override

After QA FAIL, Orchestrator reads the QA report summary. If it detects environment patterns ("docker not running", "port already in use", "permission denied"), it does NOT escalate — it routes to a fix-environment path instead of a stronger model.

```
QA failure patterns -> escalation decision:
  "docker", "port", "permission", "connection refused" -> environment failure -> don't escalate model
  "not implemented", "syntax error", "import error"   -> model gap -> escalate
  "wine", "mt5", "dll"                                -> specialized gap -> jump to tier 2
```

---

## Execution Flow

```
orchestrate stamps each slice: standard | complex | specialized

standard slice:
  attempt 1 -> forger-dev (free/default)
    PASS: complete-slice, next slice
    FAIL (env): route to environment fix, retry same tier
    FAIL (code): -> forger-dev-mid (haiku)
      PASS: complete-slice, next slice [next slice starts at free]
      FAIL: FAILED.md + ask_followup_question

complex slice:
  attempt 1 -> forger-dev-mid (haiku)   [skip free entirely]
    PASS: complete-slice, next slice
    FAIL (env): route to environment fix
    FAIL (code): -> forger-dev-high (sonnet)
      PASS: complete-slice
      FAIL: FAILED.md

specialized slice:
  attempt 1 -> forger-dev-high (sonnet) [one shot]
    PASS: complete-slice
    FAIL: FAILED.md + ask_followup_question
```

**Max paid attempts per slice: 2 for standard, 2 for complex, 1 for specialized.**
"Go back to free" is automatic — every NEW slice starts at its classified tier (standard = free).

---

## Modes Required

Current: 7 modes (forger-planner, forger-pm, forger-orchestrator, forger-dev, forger-qa, forger-debugger, forger-auditor)

New modes to add to `.kilocodemodes`:

| Mode | Model | Used When |
|------|-------|-----------|
| `forger-dev` | (no field — free/default) | standard slices, attempt 1 |
| `forger-dev-mid` | `claude-haiku-4-5-20251001` | standard attempt 2, complex attempt 1 |
| `forger-dev-high` | `claude-sonnet-4-6` | complex attempt 2, specialized attempt 1 |
| `forger-qa-mid` | `claude-haiku-4-5-20251001` | mirrors dev-mid tier |
| `forger-qa-high` | `claude-sonnet-4-6` | mirrors dev-high tier |
| `forger-debugger` | `claude-haiku-4-5-20251001` | always mid+ (analytical task) |
| `forger-debugger-high` | `claude-sonnet-4-6` | complex/specialized failures |

Total: 7 existing + 6 new = 13 modes.

---

## Files to Change

| File | Change |
|------|--------|
| `forger_cli.py` | Add `classify_slice_complexity()`, call during `cmd_orchestrate()`, stamp task packet header |
| `forger_code/data/custom_modes.yaml` | Add 6 new mode variants with model fields |
| `forger_code/data/prompts/ORCHESTRATOR.md` | Add tiered spawn logic: read Complexity-Tier, branch to correct mode, parse QA failure reason for env vs. code classification |

---

## Blocking Assumption (Must Verify First)

**The entire design depends on: does Kilo Code's `new_task()` use the mode name passed in the spawn message?**

If YES -> this works.
If NO -> all 13 modes use the same model regardless, making this useless.

**Verification test (one-off, before building):**
1. Temporarily set `forger-dev-high` with `model: claude-opus-4-6` in `.kilocodemodes`
2. Spawn a task with mode `forger-dev-high` in the message
3. Check Kilo Code UI / Anthropic console — does the task run on Opus 4.6?

If the test passes, build this. If it fails, this entire design is invalid and the fallback is "user sets model globally in Kilo Code UI."

---

## Keyword List: Starting Point (Needs Tuning)

These need calibration across 3+ real projects. Expected false positive rate initially ~10-15%.

**Specialized triggers:** `wine`, `mql5`, `mql4`, `metatrader`, `mt5`, `mt4`, `dll`, `winsock`, `winapi`, `.mqh`, `firmware`, `kernel module`, `device driver`, `cuda`, `opencl`, `simd`, `assembly`

**Complex triggers:** `docker`, `docker-compose`, `kubernetes`, `k8s`, `postgresql`, `mysql`, `mongodb`, `redis`, `kafka`, `rabbitmq`, `nginx`, `systemctl`, `migration`, `alembic`, `authentication`, `jwt`, `oauth`, `ldap`, `websocket server`, `grpc`, `protobuf`

**Standard (default):** everything else — basic Python, CRUD, config files, CLI tools, test files, simple REST endpoints

---

## Cost Impact Estimate

Based on mt5-api-docker run 2 ($12.33, 21 slices):
- ~80% of slices would classify as complex/specialized (Docker + Wine project)
- On a standard Python project: ~70% standard, 25% complex, 5% specialized
- Standard project estimated savings: ~30-40% of DEV+QA cost (65% of total) = ~20-25% total savings

For a $12 run: saves ~$2.50-3.00. Meaningful at scale, marginal for occasional use.

---

## Implementation Priority

**Not recommended for immediate implementation.** Blocking reason: the `new_task()` mode parameter is unverified.

**Recommended sequence:**
1. Finish mt5-api-docker run 3 with current architecture
2. Run the verification test above (5 minutes)
3. If test passes, implement this design
4. If test fails, document as architectural limitation

---

## Open Questions

1. Does `new_task()` mode parameter work? (Blocking — must test before building)
2. Should QA always mirror DEV tier, or can QA run one tier lower? (QA is read+evaluate, cheaper task)
3. Should Orchestrator itself run on a paid model? (Currently free; if it misclassifies failure reasons, escalation logic breaks)
4. Keyword list calibration — needs data from 3+ projects before stabilizing
