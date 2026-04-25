# api-response-standardizer

Standardize all HTTP responses and error handling in a FastAPI Clean Architecture project. Creates a unified JSON payload format for both success (2xx) and error (4xx/5xx) responses using Pydantic response models and FastAPI exception handlers.

## Trigger

Activate this skill when:
- The user asks to format errors, standardize HTTP responses, handle global exceptions, configure response middleware, or structure JSON payloads in FastAPI.
- The `/new-feature` command is being applied and standard response formatting is needed.
- The user invokes `/standardize-responses` manually.

## Input

No input required. The skill operates on the project at the current working directory.

Before starting, verify:
1. The project is a FastAPI project (check for `fastapi` in `pyproject.toml`).
2. Read `src/infrastructure/fastapi/app.py` (or equivalent main app file) to understand current global configuration.
3. Check if `src/adapters/shared/` already exists to avoid overwriting existing files.

## Process

### Step 1 - Create the Pydantic Response Schemas (`src/adapters/shared/schemas.py`)

Create standardized response schemas with two shapes:

**Success response:**
```json
{
  "status_code": 200,
  "message": "OK",
  "data": { ... }
}
```

**Error response:**
```json
{
  "status_code": 400,
  "error": "Bad Request",
  "message": ["field: value is not valid"],
  "timestamp": "2025-01-01T00:00:00Z",
  "path": "/session/start"
}
```

The schemas must:
- Use Pydantic `BaseModel` with generics for the `data` field: `ApiSuccessResponse[T]`
- Be typed with `Generic[T]` from `typing`
- Export both `ApiSuccessResponse` and `ApiErrorResponse` classes
- Include `model_config = ConfigDict(from_attributes=True)` for ORM compatibility

### Step 2 - Create the Response Middleware (`src/infrastructure/fastapi/middleware.py`)

Create a Starlette/FastAPI middleware or response wrapper that:
- Intercepts all successful JSON responses (2xx)
- Wraps the original payload inside the `data` field
- Maps HTTP status codes to human-readable messages:
  - 200 -> "OK"
  - 201 -> "Created"
  - 204 -> "No Content"
  - Default -> "Success"
- Returns the standardized `ApiSuccessResponse` shape

Prefer a **response model approach** (declaring `response_model=ApiSuccessResponse[DTO]` on each route) over blanket middleware, as it preserves FastAPI's OpenAPI schema generation.

### Step 3 - Create the Global Exception Handler (`src/infrastructure/fastapi/exception_handlers.py`)

Create FastAPI exception handlers that:

**For Pydantic `ValidationError` (422):**
- Extracts the list of validation error messages
- Returns the standardized `ApiErrorResponse` with the list in `message`

**For domain-specific exceptions:**
- Map each domain exception class to an appropriate HTTP status code
- Example: `SessionNotFoundError` -> 404, `MasteryThresholdError` -> 409
- Return a clean `ApiErrorResponse` without leaking internal details

**For unhandled exceptions (500):**
- Log the full traceback server-side
- Return a generic `ApiErrorResponse` with message `["Internal server error"]`
- Never expose stack traces or internal state to the client

Always include `timestamp` (ISO 8601 UTC) and `path` (from the request URL).

### Step 4 - Register in the FastAPI App

Show the user how to register the exception handlers globally. Apply the changes to the FastAPI app factory or `main.py`:

```python
from src.infrastructure.fastapi.exception_handlers import (
    domain_exception_handler,
    validation_exception_handler,
    unhandled_exception_handler,
)

app.add_exception_handler(DomainException, domain_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(Exception, unhandled_exception_handler)
```

### Step 5 - Show Summary

After creating all files, display:
- A list of all files created/modified
- An example success response (JSON)
- An example validation error response (JSON)
- An example domain exception response (JSON)

## Validation

1. Run type checker to verify schemas compile: `uv run mypy src/adapters/shared/ src/infrastructure/fastapi/`
2. Run linter: `uv run ruff check src/ --fix`
3. Confirm FastAPI's `/docs` (Swagger UI) still generates valid OpenAPI schemas for all routes
