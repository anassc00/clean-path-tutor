---
command: /check-arch
description: >
  Architecture sentinel for Clean Architecture compliance in Python. Triggers automatically when:
  creating a new feature, module, folder or file inside src/; proposing a new use case, entity,
  port, or adapter; detecting an import that crosses architectural layers (e.g., Domain importing
  from Infrastructure); or when the user asks about module structure, layer dependencies, or naming
  conventions. Validates against the Clean Architecture rules defined in docs/TECHNICAL_REPORT.md.
---

# architect-sentinel

Guardian of Clean Architecture compliance in the Python codebase. Validates every new module,
file, or import against the rules defined in `docs/TECHNICAL_REPORT.md`.

## Auto-Trigger Conditions

Activate this skill automatically when:
- The user creates or proposes a new feature, module, or folder inside `src/`
- The user proposes creating a new use case, entity, port, or adapter class
- An import is detected (or proposed) that may cross architectural layers
- The user asks about where to place a class, how to structure a module, or which layer something belongs to

## Process

### Phase 1 — Module Classification

Before any file is created, classify the module using the **Module Registry in §6.1** of `docs/TECHNICAL_REPORT.md`.

If the module is listed in §6.1, use that level as the source of truth. If it is not listed, classify it using the checklist from §6.2:

- Does it introduce new domain entities with invariants and a dedicated port? → **Full**
- Does it introduce domain logic and a use case but reuses existing ports? → **Standard**
- Is it a cross-cutting concern with no domain logic (auth, config, logging)? → **Light**

Output the classification result clearly:
```
Module: <name>
Assigned level: Full | Standard | Light
Reason: <brief justification referencing §6.1 or §6.2>
```

Alert if the user's proposed structure conflicts with the predefined level from §6.1.

---

### Phase 2 — Structure Validation

Based on the assigned level, enforce the folder rules from **§6.2** of `docs/TECHNICAL_REPORT.md`:

#### Full Level
```
src/
├── domain/{module}/
│   ├── __init__.py
│   ├── entities.py
│   ├── ports.py
│   └── services.py
├── application/{module}/
│   ├── __init__.py
│   ├── use_cases.py
│   └── dtos.py
├── adapters/{module}/
│   ├── __init__.py
│   ├── controllers.py
│   └── parsers.py
└── infrastructure/{module}/
    ├── __init__.py
    └── adapters.py
```

#### Standard Level
```
src/
├── domain/{module}/
│   ├── __init__.py
│   ├── entities.py
│   └── services.py
└── application/{module}/
    ├── __init__.py
    ├── use_cases.py
    └── dtos.py
```
**Rule**: Standard modules MUST NOT have `adapters/{module}/` or `infrastructure/{module}/` folders.

#### Light Level
```
src/
└── infrastructure/{module}/
    ├── __init__.py
    └── {module}.py
```
**Rule**: Light modules MUST NOT have a `domain/{module}/` or `application/{module}/` folder.

---

### Phase 3 — Naming Convention Check

Verify that all proposed or created files and classes follow the conventions from **§6.3** of `docs/TECHNICAL_REPORT.md`:

**Files:**
| Artifact | Expected filename |
|---|---|
| Domain entities | `entities.py` |
| Domain port (interface) | `ports.py` |
| Domain service | `services.py` |
| Use case | `use_cases.py` |
| Application DTO | `dtos.py` |
| Controller | `controllers.py` |
| LLM parser | `parsers.py` |
| Infrastructure adapter | `adapters.py` |

**Classes:**
| Artifact | Pattern |
|---|---|
| Domain entity | `{Concept}` (e.g., `LearningSession`) |
| Domain port | `{Concept}Port` (e.g., `TutorPort`) |
| Domain service | `{Concept}Service` (e.g., `GapAnalysisService`) |
| Use case | `{Verb}{Concept}UseCase` (e.g., `EvaluateAssessmentUseCase`) |
| Infrastructure adapter | `{Provider}{Concept}Adapter` (e.g., `LangChainOpenAITutorAdapter`) |

Flag any deviation and suggest the correct name.

---

### Phase 4 — Dependency Rule Check (§2.1)

Before any import is written, verify it complies with the **Dependency Rule** from §2.1:

```
domain/ ← application/ ← adapters/ ← infrastructure/
```

**Allowed:**
- `infrastructure/` can import from any layer
- `adapters/` can import from `application/` and `domain/`
- `application/` can import from `domain/` only
- `domain/` has zero external imports (no `langchain`, no `fastapi`, no `qdrant_client`)
- `domain/` stdlib imports allowed: `abc`, `typing`, `datetime`, `uuid` — Pydantic allowed

**Forbidden (flag immediately):**
- `domain/` importing from `application/`, `adapters/`, or `infrastructure/`
- `domain/` importing any third-party framework
- `application/` importing from `adapters/` or `infrastructure/`
- `adapters/` importing concrete classes from `infrastructure/` (only via composition root)
- `tests/unit/` importing any class from `infrastructure/`

When a violation is detected, output:
```
⚠ DEPENDENCY RULE VIOLATION (TECHNICAL_REPORT.md §2.1)
File: <file>
Illegal import: <import path>
Layer: <offending layer> → <target layer> is not allowed
Fix: Move the dependency to a Port (ABC) in domain/ or application/
```

---

### Phase 5 — Module Registry Validation (§6.1)

Consult the Module Registry table in **§6.1 of `docs/TECHNICAL_REPORT.md`**.

If the module being created is in the table:
- Confirm its assigned level matches what the table specifies
- If there is a mismatch, block the action and report:

```
⚠ MODULE LEVEL MISMATCH (TECHNICAL_REPORT.md §6.1)
Module: <name>
Expected level: <level from table>
Proposed level: <level inferred>
Action: Adjust the structure to match the predefined level.
```

---

### Phase 6 — Feedback Report

After running all phases, emit a structured compliance report before any code is written or edited:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ARCHITECT-SENTINEL REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Module       : <name>
Level        : Full | Standard | Light
Structure    : ✓ Valid | ✗ <issue>
Naming       : ✓ Valid | ✗ <issue>
Dependencies : ✓ Valid | ✗ <issue>
Module map   : ✓ Valid | ✗ <issue>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Reference    : TECHNICAL_REPORT.md §<section>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If **any check fails**, do NOT proceed with code generation. Explain the violation with the exact section of `docs/TECHNICAL_REPORT.md` that applies, and suggest the corrective action.

If **all checks pass**, confirm clearance and proceed with implementation.
