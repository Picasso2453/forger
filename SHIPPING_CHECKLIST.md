# Forger MVP Shipping Checklist

**Status:** MVP Ready
**Target:** Alpha Release v0.1.0
**Estimated Time to Ship:** 1-2 hours

---

## Pre-Shipping Tasks

### ✅ Code Quality & Documentation

- [x] **CLI Implementation** — 1,163 lines, all 16 commands working
- [x] **Agent Prompts** — 8 detailed roles, tested on 2 projects (pwgen, mt5-api-docker)
- [x] **QA Gate** — Hardened in Round 15 (infrastructure checks, test validation, no-stubs)
- [x] **Package Configuration** — pyproject.toml correct, data files bundled
- [x] **.gitignore Created** — Covers Python, IDE, OS, build artifacts
- [x] **README.md Updated** — Correct paths, workflow description, file structure
- [x] **LICENSE Created** — MIT license
- [x] **CONTRIBUTING.md Created** — Developer guide for contributors
- [x] **MVP_SHIPPING_REPORT.md** — Comprehensive analysis and recommendations

### ⏳ Pre-Publication Tasks (Do These Now)

- [ ] **Initialize Git Repository**
  ```bash
  cd c:\projects\forger-ai-selfhosted
  git init
  git add .
  git commit -m "Initial commit: Forger v0.1.0 MVP - documentation-first AI agent workflow system"
  ```
  **Time:** 2 min

- [ ] **Create GitHub Repository** (if not already done)
  - Go to https://github.com/new
  - Repository name: `forger-ai` (or similar)
  - Description: "Documentation-first, disposable-agent workflow system for AI-driven development"
  - License: MIT
  - **Time:** 5 min

- [ ] **Push to GitHub**
  ```bash
  git remote add origin https://github.com/YOUR-ORG/forger-ai.git
  git branch -M main
  git push -u origin main
  ```
  **Time:** 2 min

- [ ] **Test Installation from Source**
  ```bash
  # In a separate test directory
  mkdir test-install
  cd test-install
  python -m venv venv
  source venv/bin/activate
  pip install -e /path/to/forger-ai-repo
  forger init
  forger status
  ```
  **Time:** 10 min

- [ ] **Verify Package Data**
  ```bash
  python -c "from importlib.resources import files; print(list(files('forger_code').joinpath('data').glob('**/*')))"
  ```
  Should show all prompts and custom_modes.yaml
  **Time:** 2 min

---

## PyPI Publication Steps

### Option A: Manual Publication (Recommended for First Release)

#### Step 1: Create PyPI Account (5 min, one-time)
- Go to https://pypi.org/account/register/
- Create account with email you control
- Verify email
- Enable 2FA (highly recommended)

#### Step 2: Install Build Tools (2 min)
```bash
pip install build twine
```

#### Step 3: Build Distribution Package (2 min)
```bash
cd c:\projects\forger-ai-selfhosted
python -m build
```

Verify output:
```
dist/
├── forger-code-0.1.0.tar.gz      # Source distribution
└── forger-code-0.1.0-py3-none-any.whl  # Wheel distribution
```

#### Step 4: Verify Package Contents (3 min)
```bash
# Check source tarball
tar -tzf dist/forger-code-0.1.0.tar.gz | grep -E "(prompts|custom_modes)" | head -10

# Should show:
# forger-code-0.1.0/forger_code/data/custom_modes.yaml
# forger-code-0.1.0/forger_code/data/prompts/PLANNER_ARCHITECT.md
# ... (7 more prompt files)
```

#### Step 5: Publish to PyPI (2 min)
```bash
twine upload dist/forger-code-0.1.0*
# Enter PyPI username and token when prompted
```

#### Step 6: Verify Publication (2 min)
```bash
# Check PyPI page: https://pypi.org/project/forger-code/

# Test installation in clean environment:
python -m venv test-pypi
source test-pypi/bin/activate
pip install forger-code
forger init  # Test that it works
```

### Option B: GitHub Actions Automation (Recommended for Future Releases)

Create `.github/workflows/publish.yml`:

```yaml
name: Publish to PyPI

on:
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install build twine
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
```

Then publish via: Create a Release on GitHub → automatically publishes to PyPI

---

## Post-Publication Tasks

### Announce Release (10 min)

- [ ] **Update GitHub README section** → Link to PyPI
- [ ] **Create Release Notes** on GitHub
- [ ] **Post on relevant forums:**
  - HackerNews: Show HN: Forger — documentation-first AI workflow...
  - Reddit: r/MachineLearning, r/OpenSource
  - Twitter/X (if applicable)
- [ ] **Add to Awesome Lists:**
  - awesome-agents
  - awesome-ai-tools
  - awesome-workflow-automation

### Documentation

- [ ] **Add to README:**
  ```markdown
  ## Installation (Stable)

  ### Via pip (Recommended)
  ```bash
  pip install forger-code
  forger init
  ```

  ### From Source
  ```bash
  git clone https://github.com/YOUR-ORG/forger-ai.git
  cd forger-ai
  pip install -e .
  forger init
  ```
  ```

- [ ] **Create CHANGELOG.md** for v0.1.0:
  ```markdown
  # Changelog

  ## [0.1.0] - 2026-03-05 (MVP Alpha)

  ### Added
  - 7 agent modes (Planner, PM, Orchestrator, DEV, QA, Debugger, Auditor)
  - 16 CLI commands for project lifecycle
  - Markdown-driven workflow state system
  - QA gate with infrastructure validation
  - Wave-based parallel-ready execution model
  - Comprehensive prompt library (8 roles, 108 KB)
  - Full support for Kilo Code VSCode extension

  ### Tested
  - pwgen-cli project (2 slices, 100% pass)
  - mt5-api-docker project (21 slices across 15 rounds)

  ### Known Limitations
  - Synchronous execution assumed (async support future)
  - No adaptive model tiering yet (designed, not implemented)
  - Single project per workspace

  ### Documentation
  - README with quickstart
  - ARCHITECTURE.md overview
  - CONTRIBUTING.md developer guide
  - Comprehensive prompt documentation in source
  ```

---

## Success Criteria

After publishing, verify:

- [ ] Package appears on PyPI: https://pypi.org/project/forger-code/
- [ ] Can install: `pip install forger-code`
- [ ] Can run: `forger init` in a new directory
- [ ] README displays correctly on PyPI
- [ ] CLI help works: `forger --help`
- [ ] All 7 agent modes load correctly in Kilo Code

---

## Version Numbering Strategy

Going forward, use **Semantic Versioning**:

- **v0.1.x** (Patch) — Bug fixes, documentation
- **v0.2.0** (Minor) — New features (Adaptive Model Tiering, Meta-Analyzer)
- **v1.0.0** (Major) — Breaking changes (rare)

### Planned Releases

- **v0.1.0** — Current (MVP Alpha)
- **v0.1.1** — Bug fixes, if needed (week 1-2)
- **v0.2.0** — Adaptive Model Tiering + Meta-Analyzer (week 3-4)
- **v0.3.0** — Async execution support, web dashboard (month 2)

---

## Timeline

| Phase | Task | Time | Owner |
|-------|------|------|-------|
| **Pre-Ship** | Init git, verify package, test install | 20 min | You |
| **Publish** | Build, upload to PyPI | 10 min | You |
| **Post-Ship** | Release notes, announcements | 10 min | You |
| **Total** | | **~40 min** | |

---

## Rollback Plan

If something goes wrong after publishing:

1. **PyPI Yanking** (if critical bug):
   ```bash
   twine yank forger-code==0.1.0
   ```
   This hides the version from pip but keeps historical record.

2. **Quick Fix Release**:
   - Fix the bug
   - Bump to v0.1.1
   - Republish

3. **Documentation**:
   - Add to README: "Migrate from v0.1.0 to v0.1.1: [instructions]"

---

## Questions Before Publishing?

1. **GitHub org/username?** (for remote URL)
2. **PyPI account created?** (2FA enabled?)
3. **Package name okay?** (`forger-code` currently; could be `forger-ai`, `forger-workflow`, etc.)
4. **Any final tests needed?** (Run on test projects?)
5. **Announcement channels?** (HN, Reddit, Twitter, etc.?)

---

## Files Ready for Distribution

### Source Files (in package)
- `forger_cli.py` (45 KB)
- `forger_code/__init__.py`
- `forger_code/data/custom_modes.yaml` (24 KB)
- `forger_code/data/prompts/*.md` (8 files, 84 KB)

### Documentation (in repo, visible on GitHub)
- `README.md` (updated)
- `ARCHITECTURE.md`
- `AGENTS.md`
- `CONTRIBUTING.md`
- `LICENSE` (MIT)
- `pyproject.toml`

### Dev/Reference (in repo, not shipped)
- `IMPLEMENTATION_NOTES.md`
- `IMPROVEMENTS_TRACKER.md`
- `MVP_SHIPPING_REPORT.md`
- `SHIPPING_CHECKLIST.md` (this file)

---

## Next Steps After MVP v0.1.0

### v0.2.0 (2-4 weeks)
- [ ] Implement Adaptive Model Tiering (design doc ready)
- [ ] Implement Meta-Analyzer (design doc ready)
- [ ] CI/CD automation (GitHub Actions)

### v0.3.0+ (longer term)
- [ ] Async execution support
- [ ] Web dashboard
- [ ] MCP server integration
- [ ] Additional language support

---

**Ready to ship? Follow the "Pre-Shipping Tasks" section above, then "PyPI Publication Steps", then verify success criteria.**

**Questions?** Refer to MVP_SHIPPING_REPORT.md for detailed rationale and analysis.
