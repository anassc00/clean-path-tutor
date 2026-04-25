# doc-sync

Synchronize project documentation (CLAUDE.md and README.md) with the current source code structure, modules, and configuration.

## Trigger

Activate this skill when:
- The `/new-feature` command completes successfully
- A new folder or module is created inside `src/`
- The user asks to "update documentation", "sync docs", "actualizar la documentacion", or similar
- The user invokes `/update-docs` manually

## Process

### Step 1 - Scan source structure

Run a recursive scan of `src/` to build a complete map of existing modules and their internal folders:

```bash
find src/ -type d | sort
```

Identify each layer (`domain/`, `application/`, `adapters/`, `infrastructure/`) and the modules within each.

### Step 2 - Read architecture levels

Check if `docs/TECHNICAL_REPORT.md` exists. If it does, read §6.1 (Module Registry) to extract the architecture level assigned to each module:

| Level | Meaning |
|-------|---------|
| **Full** | All 4 layers: domain, application, adapters, infrastructure |
| **Standard** | domain + application only |
| **Light** | infrastructure only (optionally + adapters) |

If `docs/TECHNICAL_REPORT.md` does not exist, infer the level from the folder structure detected in Step 1.

### Step 3 - Update CLAUDE.md

Open `CLAUDE.md` and update the following sections. Do NOT modify sections outside of these:

#### 3a - "Estructura de Carpetas" (or equivalent folder structure section)
Regenerate the folder tree to reflect the **current** state of `src/`. Use the same indented tree format already present in the file. Include all modules with their internal layer folders and key files (e.g., `entities.py`, `use_cases.py`, `adapters.py`).

#### 3b - "Convenciones para nuevos modulos" (or equivalent conventions section)
If a new module follows a structure that differs from the documented convention, add a note or update the convention description to reflect the supported patterns.

### Step 4 - Update README.md

Open `README.md` and synchronize:

- **Project description**: Ensure it matches the current scope of the system.
- **Tech stack table**: Verify all dependencies listed are still in `pyproject.toml`. Add any new significant dependencies (e.g., new LangChain provider packages). Remove any that were removed.
- **Useful commands**: Ensure `uv run` commands listed match what is actually available:
  - `uv run pytest tests/` - run all tests
  - `uv run pytest tests/unit/` - run unit tests only
  - `uv run ruff check src/` - lint
  - `uv run mypy src/` - type check
  - `uv run uvicorn src.infrastructure.fastapi.main:app --reload` - start API server
- **Available endpoints**: If there is an endpoints section, update it to reflect current FastAPI routers and route decorators.

### Step 5 - Validate consistency

Cross-check the following between `CLAUDE.md`, `README.md`, and the actual code:

1. **FastAPI docs URL**: Confirm the documented Swagger path matches what is configured in `main.py` (default: `/docs`).
2. **Base routes**: Verify that documented API routes match the actual `@router.get/post/...` decorators in `src/adapters/*/controllers.py`.
3. **Port number**: Confirm the documented port matches the `uvicorn` configuration or `.env` variable.
4. **Python version**: Confirm the documented Python version matches the `requires-python` field in `pyproject.toml`.

If any inconsistency is found, fix it in the documentation.

### Step 6 - Report

Present a short summary to the user:

```
Documentation sync complete:
- Modules detected: [list with levels]
- CLAUDE.md sections updated: [list]
- README.md sections updated: [list]
- Inconsistencies fixed: [list or "none"]
```
