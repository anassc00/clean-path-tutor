# backend-bug-fixer

Senior Backend Architect assistant for diagnosing and fixing bugs in a Python Clean Architecture project. Analyzes stacktraces, error logs, failing endpoints, LangGraph state machine issues, Pydantic validation errors, and business logic problems. Respects the Clean Architecture dependency rule and module levels (Full/Standard/Light) defined in `docs/TECHNICAL_REPORT.md`.

## Trigger

Activate this skill when:
- The user pastes a stacktrace or error log from the FastAPI server or LangGraph agent.
- The user reports a bug, a failing endpoint, or unexpected behavior in the API.
- The user asks to fix an LLM integration issue (LangChain, LangGraph, structured output parsing).
- The user asks to solve a business logic problem in the domain or application layer.
- The user asks to fix a Pydantic validation error or type error (mypy).
- The user invokes `/fix-backend` manually.

## Process

### Step 1 - Gather Error Context

If the user has NOT provided at least one of the following, **STOP** and ask before proceeding:
- The exact error message or stacktrace
- The endpoint or agent node that is failing
- The expected vs actual behavior

Ask:
> "To diagnose this issue I need more context. Please share one of the following:
> 1. The full error message or stacktrace
> 2. Which endpoint or LangGraph node is failing
> 3. What you expected to happen vs what actually happened"

Do NOT guess or proceed without concrete error information.

### Step 2 - Analyze the Error

Once you have the error context:

1. **Parse the stacktrace** (if provided): identify the file, line number, and exception type.
2. **Read the affected files** in the codebase to understand the current implementation.
3. **Identify the root cause**: distinguish between symptoms and the actual underlying issue.
4. **Check related files**: if the bug is in a use case, also check its ports, DTOs, infrastructure adapter, and controller.

Common error categories:
- `ValidationError` from Pydantic - check entity invariants and DTO field types
- `AttributeError` / `TypeError` - check port interface vs concrete adapter signatures
- `ImportError` / `ModuleNotFoundError` - check for circular imports or dependency rule violations
- LangGraph `InvalidUpdateError` - check that node functions return valid state deltas
- LangChain structured output failures - check that the Pydantic schema matches LLM output format

Present the analysis:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BUG ANALYSIS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Error     : <exception type and message>
File      : <file path>:<line number>
Root Cause: <clear explanation of why the error occurs>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 3 - Identify the Affected Architecture Layer

Determine which Clean Architecture layer contains the bug:

| Layer | Examples |
|-------|---------|
| **Domain** | Entity validation, port ABC definition, domain service logic, domain exceptions |
| **Application** | Use case orchestration, DTO transformation, port contract usage |
| **Adapters** | FastAPI controllers, LLM output parsers, response schemas |
| **Infrastructure** | LangChain adapters, LangGraph graph builder, Qdrant adapter, FastAPI app config |

Also check if the bug violates the dependency rule (docs/TECHNICAL_REPORT.md §2.1):
```
domain/ <- application/ <- adapters/ <- infrastructure/
```

If the fix requires changes across multiple layers, list them in order from innermost (domain) to outermost (infrastructure).

### Step 4 - Apply the Fix

1. **Explain the fix** in plain language before writing code.
2. **Apply the code changes** directly to the affected files, respecting:
   - The module's architecture level (Full/Standard/Light) from `docs/TECHNICAL_REPORT.md` §6.1
   - The dependency rule: domain never imports from application or infrastructure
   - Naming conventions from §6.3 (snake_case files, PascalCase classes)
   - Existing patterns (Pydantic BaseModel entities, Port ABCs, use case execute() method)
3. **Do NOT introduce unnecessary changes** - fix only what is broken. Do not refactor surrounding code.

### Step 5 - Verify and Suggest Testing

After applying the fix:

1. **Run type checker**: `uv run mypy src/`
2. **Run linter**: `uv run ruff check src/`
3. **Run tests**: `uv run pytest tests/ -v --no-header`
4. **If tests fail**, fix them as part of the bug fix.
5. **Suggest how to manually test** the fix:
   - Provide a curl command or HTTPie request for endpoint bugs
   - Suggest a pytest unit test scenario for logic bugs
   - Suggest a LangGraph graph invocation snippet for agent node bugs

Present the result:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FIX APPLIED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Files Modified:
  - <file 1>
  - <file 2>

Type check : OK (mypy)
Linter     : OK (ruff)
Tests      : <N> passed, 0 failed

How to verify:
  <curl/httpie command or pytest snippet>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Important Rules

- This skill works exclusively on the `clean-path-tutor` Python project.
- Never introduce new dependencies to fix a bug. Use existing packages from `pyproject.toml`.
- If the bug reveals a missing test, mention it but do not create it unless asked.
- If the fix requires a new Pydantic model, place it in the correct layer (domain entities go in `src/domain/{module}/entities.py`).
- If the bug is a dependency rule violation (e.g., domain importing from infrastructure), the correct fix is to introduce a Port interface, not to suppress the import.
