# Forger Meta-Analyzer — Architecture Design Notes

## Concept

A system that analyzes Forger's own execution artifacts to identify friction points and continuously improve the workflow. The meta-analyzer reads completed project outputs, classifies failures, detects cross-project patterns, and proposes specific prompt changes for human review.

**Status:** Design / discussion phase. Not yet implemented.

---

## What Data Is Available

| Source | Signal |
|--------|--------|
| `.forger/qa_reports/[id].md` | Failure patterns, specific criteria that keep failing |
| `.forger/tasks/[id]-DONE.md` | DEV self-assessment accuracy (did DEV think it passed but QA disagreed?) |
| `.forger/tasks/[id]-DEBUG.md` | Root causes the Debugger identified, which fix option was chosen |
| `.forger/tasks/[id]-FAILED.md` | Fail-save cases — the hardest, most persistent failures |
| `.forger/AUDIT.md` | Systemic protocol violations |
| `.forger/SLICES.md` status progression | STALE patterns, which slices needed debug cycles |

**Known blind spot:** Chat/task logs are not captured in artifacts. Without them, we can see that a failure happened but not the reasoning process that led to it. This limits attribution accuracy.

---

## Proposed Architecture

### Layer 1 — Collector
Reads all `.forger/**` artifacts from one or more completed projects. Builds a structured event log per slice:

```
[slice_id, qa_attempts, debug_cycles, final_status, failure_reasons, stale_count, ...]
```

### Layer 2 — Friction Classifier
Categorizes each failure by type:

| Category | Description |
|----------|-------------|
| Contract quality failure | Vague criteria → DEV misunderstood |
| DEV scope creep | Created files not listed in contracts |
| Test discovery failure | "0 collected", wrong test path |
| Stub/simulation substitution | In-memory data instead of real integration |
| Wrong file path | File created at wrong location |
| PM planning error | Wrong field names, missing contracts, bad slice format |
| External dependency failure | Environment issue, not a prompt problem |
| Model capability limit | Task beyond current model; not fixable via prompts |

### Layer 3 — Pattern Aggregator
Cross-project analysis. Separates signal from noise:

- **N >= 3 occurrences** of the same failure category across different projects = systemic signal
- **N < 3** = potentially project-specific or environment-specific; flag but don't act

Example output:
```
"Test discovery failures: 5 occurrences across 3 projects (S3, S7, S12 in mt5; S2, S8 in pwgen)"
vs
"Wine/MT5 startup timeout: 1 occurrence (mt5-api-docker only — environment-specific)"
```

### Layer 4 — Prompt Gap Identifier
Maps classified failures to specific prompt sections:

```
Test discovery failures       --> DEV.md Step 5 (test execution instructions)
Vague contract criteria       --> PM.md Step 5 (criteria quality rules)
DEV scope creep               --> DEV.md Step 3 (critical guard before creating files)
PM missing contracts          --> PM.md Step 7 (quality check / contract count check)
Orchestrator loop confusion   --> ORCHESTRATOR.md (authoritative source conflict)
```

### Layer 5 — Improvement Proposal Generator
Produces structured diff proposals. Each proposal contains:

- **Target**: which file, which section
- **Proposed change**: specific text diff (not vague "improve this section")
- **Evidence**: list of projects + slice IDs that triggered this
- **Confidence**: `low` (1-2 occurrences) / `medium` (3-5) / `high` (6+)
- **Expected outcome**: which failure category this would prevent

### Layer 6 — Human Review Gate (non-negotiable)
Present proposals to user → user approves or rejects each individually.

**Only after explicit approval:**
1. Archive old prompt version (`forger_code/data/prompts/history/PM.md.vN`)
2. Apply the change
3. Log the improvement with evidence trail

---

## Key Safeguards

### 1. Never Auto-Apply
The review gate is architecturally mandatory. The system identifies and proposes only — it never modifies prompts autonomously.

### 2. Cross-Project Signal Threshold
A single project's failure (especially a complex or unusual one like mt5-api-docker) may be environment-specific. N >= 3 occurrences of the same failure *category* across different projects before treating it as a prompt deficiency.

### 3. Failure Attribution Problem
The hardest problem. A failure could be caused by:
- A **prompt gap** — fixable by improving instructions
- A **model capability limit** — not fixable via prompts
- An **environment issue** — not Forger's fault
- An **ambiguous project requirement** — PM/Planner issue, not DEV/QA

A naive analyzer would blame prompts for everything. Attribution rules must be explicit.

### 4. Versioned Prompt History
Every applied change stores the previous version:
```
forger_code/data/prompts/history/PM.md.v14
forger_code/data/prompts/history/PM.md.v15
```
Rollback must be trivial (copy file back, no git required).

### 5. Scope Boundary
The analyzer proposes changes to **prompt text only**. It never proposes changes to:
- `forger_cli.py` behavior or architecture
- SLICES.md format or contract schema
- Execution flow or CLI commands

Those require deliberate human engineering decisions.

### 6. Separation from Active Projects
The meta-analyzer runs post-completion only. Never during an active execution — no interference with in-progress agents.

### 7. Local Optimum Risk
Tuning prompts on one specific project type (e.g., Docker/Wine) might improve Forger for that domain while degrading performance on simpler projects. Cross-domain test coverage is needed before applying changes.

---

## Implementation Options

### Option A — Retrospective CLI Command (Recommended Starting Point)

```bash
forger analyze [project-path]                    # analyze one completed project
forger analyze --cross [dir-containing-projects] # cross-project pattern detection
```

Output: `FRICTION_REPORT.md` in the analyzed project (or a central location)

**Advantages:** Minimal surface area, no auto-modification risk, fits existing CLI pattern, produces human-readable evidence for manual decisions.

### Option B — Forger-Meta Agent Mode

A new Kilo Code mode (`forger-meta-analyzer`) that reads artifacts and produces proposals using an AI agent.

**Advantages:** Can reason about patterns more flexibly than a deterministic classifier.

**Disadvantages:** The meta-analyzer has the same failure modes as any other agent — it can hallucinate patterns or misattribute failures. Requires the same human review gate regardless.

### Option C — CI-Style Post-Project Job

Runs automatically after each project completes (triggered by `ALL_DONE` in next-wave output).

**Disadvantages:** Highest complexity, hardest to debug, most risk of interference.

**Recommendation:** Start with Option A. It gives you evidence for human decisions without creating new failure modes. Option B is viable after Option A has produced reliable signal for a few projects.

---

## FRICTION_REPORT.md Format (Draft)

```markdown
# FRICTION REPORT — [Project Name]
Generated: [YYYY-MM-DD]

## Summary
- Total slices: N
- Slices that required debug cycles: N (N%)
- Slices that hit fail-save (3x failure): N
- STALE slices: N

## Failure Events

### [Slice ID] — [Slice Title]
- QA attempts: N
- Failure categories: [test_discovery, contract_quality, ...]
- Debug cycles: N
- Final status: done / failed
- Root cause (from DEBUG.md): [excerpt]

[repeat for all slices with failures]

## Pattern Summary

| Failure Category | Count | Projects Affected | Confidence |
|-----------------|-------|-------------------|------------|
| Test discovery failure | 5 | mt5-api-docker, pwgen | high |
| Contract quality failure | 2 | mt5-api-docker | low |

## Proposed Improvements

### Proposal 1 — [Short title]
Target: DEV.md, Step 5
Evidence: [slice IDs and projects]
Confidence: high
Proposed change:
  OLD: [current text excerpt]
  NEW: [proposed text excerpt]
Expected outcome: Prevents test discovery failures where pytest reports "0 collected"

[repeat for each proposal]

## Attribution Notes
[Any failures classified as environment-specific or model-capability-limited, excluded from proposals]
```

---

## Why mt5-api-docker Is the Best Learning Case

This project stress-tests multiple failure dimensions simultaneously:

- **Long dependency chain**: Docker + Wine + MT5 + REST + WS + noVNC — if any earlier slice fails, everything downstream blocks
- **Environment specificity**: Wine/MT5 container setup has limited model training signal; contracts must compensate
- **Contract granularity stress test**: PM must decompose a complex system into atomic slices; badly designed slices show up as STALE or repeated debug cycles
- **Test realism boundary**: "Does MT5 actually start in Wine" is not `pytest`-verifiable without real infrastructure; QA is tested on how it handles that boundary
- **DAG stress test**: The dependency graph for this project is one of the deepest we've run

Failure artifacts from this project are the highest-value training data we have.

---

## Open Questions

1. **Chat log access**: Can Kilo Code expose task conversation logs as files? Without this, attribution is limited to output artifacts only.
2. **False positive rate**: How often will the classifier attribute a model-capability failure to a prompt gap? This needs calibration across multiple projects.
3. **Storage location**: Does `FRICTION_REPORT.md` live in the analyzed project's `.forger/` directory, or in a central `forger-ai-selfhosted/reports/` location?
4. **Cross-project state**: If analyzing multiple projects, where does the aggregated pattern data live?
5. **Minimum project corpus**: How many completed projects are needed before cross-project patterns are meaningful? Likely 3-5 minimum.
