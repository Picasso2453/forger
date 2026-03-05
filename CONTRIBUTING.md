# Contributing to Forger

Thank you for your interest in contributing! Forger is an open-source AI agent workflow system, and contributions from the community help make it better.

## Getting Started

### Local Development Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/[your-org]/forger-ai.git
   cd forger-ai
   ```

2. **Create a virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install in editable mode**
   ```bash
   pip install -e .
   ```

4. **Verify installation**
   ```bash
   forger --help
   ```

## Making Changes

### Code Structure

- **forger_cli.py** — Main CLI with all commands and business logic
- **forger_code/data/custom_modes.yaml** — Agent mode definitions for Kilo Code
- **forger_code/data/prompts/** — 8 detailed role instruction files

### What to Modify

#### Updating Agent Prompts

Edit files in `forger_code/data/prompts/`:
- `PLANNER_ARCHITECT.md` — Discovery session logic
- `PM.md` — Planning and decomposition
- `ORCHESTRATOR.md` — Execution loop coordination
- `DEV.md` — Implementation instructions
- `QA.md` — Validation criteria
- `DEBUGGER.md` — Issue diagnosis and fixing
- `AUDITOR.md` — Compliance review
- `[others]`

**Testing:** After changes, run `forger init` in a test project and test the affected agent mode.

#### Updating CLI Commands

Edit `forger_cli.py`:
- Add new functions (`cmd_mycommand()`)
- Register in the main argument parser
- Add to `HELP_TEXT`

#### Updating Agent Modes

Edit `forger_code/data/custom_modes.yaml`:
- Kilo Code reads this file to populate agent definitions
- Each mode includes: name, description, roleDefinition, groups, customInstructions

### Testing Your Changes

1. **Create a test project**
   ```bash
   mkdir test-project-001
   cd test-project-001
   forger init
   ```

2. **Test the affected agent mode**
   - Open in VSCode with Kilo Code extension
   - Switch to the relevant agent mode (e.g., `forger-planner`)
   - Run through a typical workflow

3. **Check the output**
   - Verify `.forger/` directory structure
   - Check state files (PROJECT.md, STATE.md, SLICES.md)
   - Confirm agent output matches expectations

### Example: Adding a New CLI Command

```python
def cmd_myfeature(args):
    """Implement my new feature."""
    # Your logic here
    print("Feature executed successfully")

# In main():
parser.add_parser("myfeature", help="Description of my feature")
# ...
elif args.command == "myfeature":
    cmd_myfeature(args)
```

Then test:
```bash
forger myfeature
```

## Commit Guidelines

1. **Write clear commit messages**
   ```
   Fix: QA validation for empty test collections

   - Added check for "0 collected" in test output
   - Treats as FAIL per contract requirements
   - Prevents silent test skips
   ```

2. **Reference issues**
   ```
   Fixes #42: Adaptive model tiering support
   ```

3. **Keep commits focused**
   - One feature or fix per commit
   - Test before committing

## Pull Request Process

1. Create a feature branch
   ```bash
   git checkout -b feature/my-feature
   ```

2. Make your changes and test thoroughly

3. Commit with clear messages

4. Push to your fork
   ```bash
   git push origin feature/my-feature
   ```

5. Create a Pull Request with:
   - Clear description of changes
   - Reference to related issues
   - Testing performed

6. Address review feedback

## Reporting Issues

Use GitHub Issues with tags:
- `bug` — Something is broken
- `feature-request` — New functionality
- `documentation` — Docs need update
- `question` — Need clarification

Include:
- Forger version (`forger --version`)
- Python version (`python --version`)
- Steps to reproduce (for bugs)
- Expected vs. actual behavior

## Design Philosophy

Before making significant changes, understand Forger's core principles:

1. **Documentation-first**: All state lives in Markdown files
2. **Disposable agents**: No memory between tasks; agents are stateless
3. **Deterministic**: Same inputs → same outputs
4. **Auditable**: Full execution trail in `.forger/` directory
5. **Markdown as source of truth**: Not agent memory or databases

See `ARCHITECTURE.md` for more details.

## Key Files for Understanding

- **IMPLEMENTATION_NOTES.md** — 15 rounds of implementation history, decisions, and bug fixes
- **IMPROVEMENTS_TRACKER.md** — 32+ identified issues across development rounds
- **ARCHITECTURE.md** — System design overview
- **docs/design/** — Future features (Adaptive Model Tiering, Meta-Analyzer)

## Questions?

- Check existing issues and discussions
- Read IMPLEMENTATION_NOTES.md for historical context
- Review the prompt files in `forger_code/data/prompts/` to understand agent behavior

Thank you for contributing!
