# Forger — Packaging Options

## Context

Forger currently requires the user to run a Python script by absolute path to initialize each project:
```bash
python c:\projects\forger-ai-selfhosted\forger_cli.py init
```
This is friction. The goal is a single install that makes `forger` available everywhere.

---

## Option A — Python Package on PyPI *(implemented)*

**Install:**
```bash
pip install forger-code
```

**Per-project setup:**
```bash
cd my-project
forger init
```

**How it works:**
- `pyproject.toml` packages `forger_cli.py` + data files as a Python package
- `forger` console script entry point is registered at install time
- `forger_code/` Python package contains all data (prompts, modes) accessible via `importlib.resources`
- `forger init` copies `forger_cli.py` and `.kilocodemodes` to the project root (backward compat for agents using `python forger_cli.py`)
- `forger install` writes Kilo Code modes to a global location (once per machine)

**Trade-offs:**
- Pro: Minimal implementation effort on existing code
- Pro: Works on all platforms with Python 3.11+
- Pro: Standard tooling (`pip install`, `pip upgrade`, `pip uninstall`)
- Con: User must still run `forger init` per project (can be auto-triggered by Planner)
- Con: Requires Python to be in PATH

---

## Option B — MCP Server *(deferred)*

Expose all Forger state operations as MCP (Model Context Protocol) tools that Kilo Code calls natively, without `execute_command`.

**Install:**
```bash
npm install -g forger-mcp
# or
pip install forger-mcp
```

**How it works:**
- Kilo Code connects to Forger as an MCP server
- Agents call `forger_init()`, `forger_lock()`, `forger_next_slice()` as typed tool calls
- No stdout parsing — returns structured JSON
- No `execute_command` needed in any agent

**Trade-offs:**
- Pro: Most robust agent integration (typed returns, no string parsing)
- Pro: Works with any MCP-compatible AI host (not just Kilo Code)
- Pro: No `python forger_cli.py` in prompts — cleaner agent instructions
- Con: Requires building an MCP server (TypeScript or Python with `mcp` SDK)
- Con: User must configure MCP connection in Kilo Code settings
- Con: More complex architecture

**Estimated effort:** ~1 week

---

## Option C — VSCode Extension *(future vision)*

Full native VSCode extension with:
- Command palette: `Forger: Initialize`, `Forger: Lock`, `Forger: Status`
- Status bar showing current phase and slice progress
- Sidebar with slice list, contract viewer, audit report
- CLI rewritten in TypeScript — no Python dependency

**Trade-offs:**
- Pro: Best user experience — no terminal needed for setup
- Pro: No Python required
- Pro: Deep VSCode integration (file decorations, inline progress, etc.)
- Con: Largest implementation effort (2-4 weeks)
- Con: Requires publishing to VS Marketplace
- Con: Must keep Kilo Code integration working alongside extension UI

**Estimated effort:** 2-4 weeks

---

## Recommendation

1. **Now:** Option A (pip package) — half-day implementation, eliminates path friction
2. **Next:** Option B (MCP server) — once workflow is stable, remove `execute_command` dependency
3. **Long-term:** Option C (VSCode extension) — after MCP proves stable
