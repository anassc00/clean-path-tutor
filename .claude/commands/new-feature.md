---
command: /new-feature
description: >
  Full-cycle feature orchestrator for Python Clean Architecture. Triggers automatically when the
  user wants to create a new feature, implement a requirement, build new functionality, or develop
  a new use case. Automates the complete Red-Green-Refactor TDD cycle, coordinating architecture
  validation (/check-arch), test generation (/test-gen), implementation, and test verification.
  Ensures Clean Architecture compliance at every step.
---

# feature-orchestrator

Automates the complete development cycle of a feature following **Red-Green-Refactor TDD**,
ensuring compliance with the Clean Architecture and dependency rules defined in
`docs/TECHNICAL_REPORT.md`. Orchestrates the other skills (`/check-arch`, `/test-gen`) as part
of the pipeline.

## Auto-Trigger Conditions

Activate this skill automatically when:
- The user says "create a new feature", "implement a requirement", "build new functionality"
- The user describes a feature they want developed end-to-end
- The user invokes `/new-feature` manually

## Process

### Step 1 - Capture Requirements

Ask the user two questions before proceeding:

1. **Feature**: "Describe the feature you want to implement."
2. **Module**: "Which module does this feature belong to? (e.g., `session`, `assessment`, `concept`)"

Wait for both answers. Do not proceed until both are provided.

Output:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FEATURE ORCHESTRATOR - INIT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Feature : <description>
Module  : <module name>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Starting pipeline...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### Step 2 - Architecture Validation (Sentinel)

Invoke the **`/check-arch`** skill logic to:
- Determine the module's architecture level (Full / Standard / Light) from `docs/TECHNICAL_REPORT.md` section 6.1
- Validate that the folder structure exists and is correct for that level (section 6.2)
- Confirm naming conventions (section 6.3) and import boundary rules (section 6.4)

**Gate**: If `/check-arch` reports any violation, **STOP** the pipeline. Fix the structural issue first before proceeding. Do not generate tests or implementation code on an invalid structure.

Output after passing:
```
[Step 2/6] - Architecture validated - Level: <Full|Standard|Light>
```

---

### Step 3 - Phase RED: Generate Tests (TDD)

Invoke the **`/test-gen`** skill logic to:
- Identify the correct layer (Domain or Application) based on the feature description
- Generate the `test_*.py` file in the correct path following section 6.3 naming conventions
- Include at least 1 happy path and 1 error case with proper mocks via `unittest.mock`

Output after generation:
```
[Step 3/6] - Test file created - <path/to/test_file.py>
```

---

### Step 4 - Phase RED: Verify Tests Fail

Run the generated test to confirm it fails (RED phase of TDD):

```bash
uv run pytest <path/to/test_file.py> -v --no-header
```

**Gate**: The test **MUST fail**. This confirms the tests are valid and the implementation does not exist yet.

- If tests **FAIL** - proceed to Step 5
- If tests **PASS** - something is wrong. Investigate why the tests pass without implementation and fix the test file before continuing.

Output:
```
[Step 4/6] - RED phase confirmed - Tests fail as expected
```

---

### Step 5 - Phase GREEN: Implement the Code

Write the implementation code to make the tests pass:

- **Domain layer** features - create or update `entities.py`, `ports.py`, or `services.py` inside `src/domain/{module}/`
- **Application layer** features - create or update `use_cases.py` and `dtos.py` inside `src/application/{module}/`
- **Full-level modules** - also create `controllers.py` in `src/adapters/{module}/` and `adapters.py` in `src/infrastructure/{module}/`

**Implementation rules:**
- Domain code (`src/domain/`) must have **zero external imports** (no `langchain`, no `fastapi`, no `qdrant_client`)
- Domain imports are limited to: `pydantic`, `abc`, Python stdlib, and other `src/domain/` modules
- Application code may only import from `src/domain/` (ports, entities, services)
- Follow the naming conventions from `docs/TECHNICAL_REPORT.md` section 6.3
- Never raise generic exceptions in domain - define specific domain exceptions as subclasses of `Exception`

After writing the implementation, run the test again:

```bash
uv run pytest <path/to/test_file.py> -v --no-header
```

**Gate (CRITICAL)**: Tests **MUST pass**. This is a hard gate.

- If tests **PASS** - proceed to Step 6
- If tests **FAIL** - analyze the error output, fix the implementation, and re-run. **Repeat until all tests pass.** Do NOT advance to Step 6 with failing tests.

Output:
```
[Step 5/6] - GREEN phase confirmed - All tests pass
```

---

### Step 6 - Phase REFACTOR and Verify

Once tests are green, perform cleanup:

#### 6a. Run Linter and Formatter
```bash
uv run ruff check src/ --fix
uv run ruff format src/
```

Fix any linting issues. Re-run tests after to confirm nothing broke.

#### 6b. Run Type Checker
```bash
uv run mypy src/
```

Resolve any type errors. All new code must pass mypy with no errors.

#### 6c. Final Full Test Run

Run the complete test suite to ensure no regressions:
```bash
uv run pytest tests/ -v --no-header
```

---

### Step 7 - Pipeline Summary

Output the final report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FEATURE ORCHESTRATOR - COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Feature      : <description>
Module       : <module name>
Level        : Full | Standard | Light
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Architecture : Validated
Test file    : <path>
Impl files   : <paths>
Tests        : All passing (<count> tests)
Linter       : Clean (ruff)
Types        : Clean (mypy)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TDD Cycle    : RED - GREEN - REFACTOR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
