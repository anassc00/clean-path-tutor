---
command: /test-gen
description: >
  TDD test generator for Python Clean Architecture. Triggers automatically when the user requests
  a new feature, new functionality, business logic implementation, use case, or domain entity.
  Generates unit test files (test_*.py) BEFORE the implementation code, following TDD methodology.
  Respects the module's architecture level (Full/Standard/Light) defined in docs/TECHNICAL_REPORT.md
  and enforces layer isolation and dependency rules in test code.
---

# tdd-tester

Generates unit test files (`test_*.py`) **before** implementation code, following a strict TDD
approach. Tests are tailored to the module's Clean Architecture level (Full / Standard / Light)
as defined in `docs/TECHNICAL_REPORT.md`.

## Auto-Trigger Conditions

Activate this skill automatically when:
- The user requests a "new feature", "new functionality", or "business logic"
- The user describes a use case, domain rule, or entity to implement
- The user invokes `/test-gen` manually

## Process

### Step 1 — Consult Architecture Reference

Read `docs/TECHNICAL_REPORT.md` §6.1 to identify the **architecture level** of the target module:
- **Full**: Has `domain/`, `application/`, `adapters/`, `infrastructure/` layers
- **Standard**: Has `domain/` and `application/` layers only
- **Light**: Has only `infrastructure/` (no domain layer)

Output:
```
Module: <name>
Architecture level: Full | Standard | Light
Source: TECHNICAL_REPORT.md §6.1
```

If the module is not listed in §6.1, apply the classification checklist from §6.2 to determine the level before proceeding.

---

### Step 2 — Identify Target Layer

Based on the feature description, determine which layer the test belongs to:

| Feature type | Target layer | Test location |
|---|---|---|
| Domain invariants, entity validation, value objects, domain services | **Domain** | `tests/unit/domain/{module}/` |
| Use case orchestration, port coordination, DTO transformation | **Application** | `tests/unit/application/{module}/` |

**Rule**: If the module is **Light**, there is no domain layer. Tests go in `tests/unit/` directly against the infrastructure component, treating it as an integration test boundary.

---

### Step 3 — Determine File Path

Build the test file path following the naming convention from §6.3:

**Domain test:**
`tests/unit/domain/{module}/test_{module}_{artifact}.py`
Example: `tests/unit/domain/assessment/test_assessment_entities.py`

**Application test:**
`tests/unit/application/{module}/test_{module}_{use_case_name}.py`
Example: `tests/unit/application/assessment/test_assessment_evaluate_assessment_use_case.py`

Output the planned path before creating:
```
Test file: tests/unit/<layer>/<module>/test_<name>.py
```

---

### Step 4 — Generate Test with Proper Isolation

Create the test file using **pytest** with the following rules:

#### Mocking Strategy
- Mock all **Port interfaces** (ABCs) using `unittest.mock.MagicMock()` or `pytest-mock`'s `mocker.MagicMock()`
- Never instantiate real infrastructure implementations (`LangChainOpenAITutorAdapter`, `QdrantContentRepositoryAdapter`, etc.)
- Inject mocked ports into use case constructors (constructor injection pattern)
- Use `spec=PortClass` when creating mocks to enforce interface contracts: `MagicMock(spec=TutorPort)`

#### Golden Rule for Domain Tests
Domain layer tests must **NEVER** import:
- `langchain`, `langgraph`, `fastapi`, `qdrant_client`, or any third-party framework
- Anything from `src/adapters/` or `src/infrastructure/`

Domain tests should only import:
- The entity/service/port being tested (from `src/domain/{module}/`)
- Python stdlib (`pytest`, `unittest.mock`)

#### Golden Rule for Application Tests
Application layer tests must **NEVER** import:
- Concrete infrastructure implementations (e.g., `LangChainOpenAITutorAdapter`)
- `fastapi`, `langchain`, `qdrant_client`, or any framework

Application tests may import:
- The use case being tested (from `src/application/{module}/use_cases.py`)
- Domain entities and port interfaces for mock spec (from `src/domain/{module}/`)
- `unittest.mock.MagicMock`, `pytest`

---

### Step 5 — Test Structure

Every generated test file must follow this structure:

```python
import pytest
from unittest.mock import MagicMock

from src.domain.{module}.ports import {PortClass}       # for mock spec
from src.application.{module}.use_cases import {UseCase}
from src.application.{module}.dtos import {InputDTO}


class Test{UseCase}:
    def setup_method(self):
        """Initialize mocks and SUT before each test."""
        self.mock_{port} = MagicMock(spec={PortClass})
        self.sut = {UseCase}({port}=self.mock_{port})

    # --- Happy Path ---

    def test_should_{expected_happy_path_behavior}(self):
        # Arrange
        self.mock_{port}.{method}.return_value = {expected_return}
        input_dto = {InputDTO}(...)

        # Act
        result = self.sut.execute(input_dto)

        # Assert
        assert result.{field} == {expected_value}
        self.mock_{port}.{method}.assert_called_once_with(...)

    # --- Error Cases ---

    def test_should_raise_{exception}_when_{error_condition}(self):
        # Arrange
        self.mock_{port}.{method}.side_effect = {DomainException}(...)
        input_dto = {InputDTO}(...)

        # Act & Assert
        with pytest.raises({DomainException}):
            self.sut.execute(input_dto)
```

**Requirements:**
- Always include at least **1 happy path** test
- Always include at least **1 error case** raising a domain-specific exception
- Use `self.sut` (System Under Test) as the variable name for the class being tested
- Use **Arrange / Act / Assert** pattern with comments in every test
- Use `spec=PortClass` on every mock to catch interface violations at test time
- Use descriptive `test_should_*` method names that describe expected behavior

---

### Step 6 — Compliance Report

After generating the test file, output a compliance summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TDD-TESTER REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Module       : <name>
Level        : Full | Standard | Light
Layer        : Domain | Application
Test file    : <path>
Happy paths  : <count>
Error cases  : <count>
Mocks        : <list of mocked port interfaces>
Domain purity: ✓ No infrastructure imports
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Next step    : Run `uv run pytest <path> -v` — it should FAIL (RED phase)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Remind the user: **"Tests are ready. Run them to confirm RED, then implement the code to make them pass."**
