# Technical Report: clean-path-tutor

## 1. Executive Summary

`clean-path-tutor` is an interactive mentorship system driven by AI agents. Its core function is to guide learners through a structured, adaptive knowledge acquisition process: it evaluates the learner's current understanding, identifies conceptual gaps, delivers targeted instructional content, and iterates until a predefined mastery threshold is met.

The system is not designed as a monolithic LLM wrapper. It is architected as a stateful agent graph where each node encapsulates a discrete pedagogical action (assessment, diagnosis, instruction, evaluation), and transitions between nodes are governed by explicit domain logic. This separation is intentional: the decision of *what* to teach next is a domain concern; the decision of *how to invoke a model* to generate that content is an infrastructure concern.

The architectural mandate for this project is strict adherence to Clean Architecture and SOLID principles. The domain layer must be independently testable without instantiating a single LLM client, database connection, or HTTP handler. All external dependencies flow inward via dependency inversion; the domain defines the contracts, infrastructure fulfills them.

---

## 2. Architectural Principles (Clean Architecture & SOLID)

### 2.1 The Dependency Rule

The single governing constraint of Clean Architecture is the **Dependency Rule**: source code dependencies can only point inward. Nothing in an inner layer knows anything about an outer layer. Specifically:

- `domain/` has zero knowledge of `application/`, `adapters/`, or `infrastructure/`.
- `application/` depends on `domain/` abstractions only — never on concrete infrastructure implementations.
- `adapters/` translates between the external world and `application/` use cases.
- `infrastructure/` holds every concrete implementation and wires the dependency graph together.

Violations of this rule (e.g., importing a `LangChain` class directly inside a use case) would couple the business logic to a specific vendor, making it impossible to test the use case without a live model and breaking the system's ability to swap providers without touching domain or application code.

### 2.2 SOLID Principles Applied

| Principle | Application in this project |
|---|---|
| **Single Responsibility** | Each use case class in `application/` handles exactly one orchestration flow. Each domain entity models exactly one concept (e.g., `LearningSession`, `KnowledgeGap`, `MasteryScore`). |
| **Open/Closed** | New LLM providers or vector store backends are added by implementing existing interfaces — not by modifying use cases. |
| **Liskov Substitution** | All concrete adapters (e.g., `OpenAITutorAdapter`, `AnthropicTutorAdapter`) are substitutable for their abstract base class `TutorPort` without altering the behavior of the calling use case. |
| **Interface Segregation** | Ports are narrow and role-specific. A `ContentGeneratorPort` is separate from an `AssessmentEvaluatorPort`. No concrete class is forced to implement methods it does not use. |
| **Dependency Inversion** | Use cases declare dependencies on abstract `Port` interfaces. Concrete implementations are injected at the composition root in `infrastructure/`. |

### 2.3 Ports & Adapters Pattern

The boundary between `application/` and `infrastructure/` is enforced through the **Ports & Adapters** (Hexagonal Architecture) pattern. The application layer defines abstract Python interfaces (`ABCs`) — these are the **Ports**. The infrastructure layer provides concrete classes that implement those interfaces — these are the **Adapters**. No use case ever imports from `langchain`, `qdrant_client`, or `fastapi` directly.

---

## 3. Directory Structure Analysis

```
clean-path-tutor/
├── src/
│   ├── domain/
│   ├── application/
│   ├── adapters/
│   └── infrastructure/
├── tests/
│   ├── unit/
│   └── integration/
├── docs/
├── .gitignore
├── pyproject.toml
└── README.md
```

### 3.1 `src/domain/`

**Role:** The innermost ring. Contains pure business logic with zero external dependencies.

**Contents:**
- **Entities**: Pydantic models representing core domain objects — `LearningSession`, `Concept`, `KnowledgeGap`, `AssessmentResult`, `MasteryScore`. These carry domain invariants enforced via Pydantic validators (e.g., `MasteryScore` must be in `[0.0, 1.0]`).
- **Value Objects**: Immutable objects without identity (e.g., `DifficultyLevel`, `LearningObjective`).
- **Domain Interfaces (Ports)**: Abstract Base Classes (`ABCs`) that define what the domain *needs* from the outside world without specifying *how* those needs are met. Examples: `TutorPort`, `ContentRepositoryPort`, `AssessmentEvaluatorPort`.
- **Domain Services**: Pure functions or stateless classes that implement domain logic requiring collaboration between multiple entities (e.g., `GapAnalysisService`).

**Dependency constraint:** `import` statements in this layer are limited to the Python standard library and `pydantic`. No third-party framework imports are permitted.

### 3.2 `src/application/`

**Role:** Orchestration layer. Contains use cases that coordinate domain objects and call domain ports to fulfill a specific business flow.

**Contents:**
- **Use Cases**: Classes with a single `execute()` method. Examples: `StartLearningSessionUseCase`, `EvaluateAssessmentUseCase`, `GenerateExplanationUseCase`.
- **Application Services**: Cross-cutting orchestration logic that does not belong to a single use case.
- **Application DTOs**: Input/output data transfer objects specific to use case boundaries (distinct from API-level DTOs).

**Dependency constraint:** This layer may import from `domain/`. It references `domain/` ports by their abstract interface only — never by a concrete implementation class. Concrete implementations are injected via constructor.

### 3.3 `src/adapters/`

**Role:** Translation layer between the external world and the application boundary.

**Contents:**
- **Input Adapters (Controllers)**: Translate raw HTTP requests (FastAPI route handlers) or CLI inputs into application-layer DTOs and invoke the appropriate use case.
- **Output Adapters (Presenters)**: Transform use case output DTOs into HTTP response schemas.
- **LLM Response Parsers**: Structured output parsers that convert raw LLM completions into typed domain objects.

**Dependency constraint:** This layer imports from `application/` and `domain/`. It does not import from `infrastructure/` directly — infrastructure implementations are injected at the composition root.

### 3.4 `src/infrastructure/`

**Role:** The outermost ring. Contains all concrete implementations, framework integrations, and external service clients.

**Contents:**
- **LLM Adapters**: Concrete implementations of `TutorPort` using LangChain runnables (e.g., `LangChainOpenAITutorAdapter`, `LangChainAnthropicTutorAdapter`).
- **Vector Store Adapters**: Concrete implementations of `ContentRepositoryPort` using Qdrant.
- **LangGraph State Graph**: The stateful agent orchestration graph. Nodes map to use cases; edges encode the pedagogical state machine.
- **FastAPI Application**: Router definitions, dependency injection wiring, lifespan management.
- **Composition Root**: The single location where abstract ports are bound to concrete implementations and injected into use cases.
- **Configuration**: Environment variable parsing via Pydantic `BaseSettings`.

**Dependency constraint:** This layer may import from all inner layers and from any third-party library. It is the only layer with unrestricted external dependencies.

### 3.5 `tests/`

The test directory mirrors the architectural split:

- `tests/unit/`: Tests for `domain/` and `application/`. All ports are replaced with `unittest.mock` or hand-written fakes. No LLM calls, no database connections, no HTTP servers. These tests verify business logic in isolation and must run in milliseconds.
- `tests/integration/`: Tests for `adapters/` and `infrastructure/`. These tests verify that concrete adapters correctly implement their port contracts and that the LangGraph state machine transitions through nodes correctly given real (or containerized) external services.

---

## 4. Recommended Technology Stack & Infrastructure Drivers

All technology choices below are confined to the `infrastructure/` layer. The domain and application layers are agnostic to every tool listed here.

### 4.1 Dependency Management: `uv`

**Rationale:** `uv` replaces `pip`, `pip-tools`, and `virtualenv` with a single, Rust-based tool. It resolves and installs dependencies an order of magnitude faster than `pip` and produces a `uv.lock` file that guarantees byte-for-byte reproducible environments across development machines and CI runners.

For a project where the `pyproject.toml` defines the full dependency graph (including optional groups for `dev`, `test`, `docs`), `uv sync` is the single command that brings any environment to a known state. This eliminates the "works on my machine" class of infrastructure failures.

**Relevance to decoupling:** Reproducible environments mean that swapping an LLM adapter (e.g., replacing `langchain-openai` with `langchain-anthropic`) is a clean dependency graph change, not an environment archaeology exercise.

### 4.2 Entities & Validation: `Pydantic` v2

**Rationale:** Pydantic v2 (Rust-backed core) is used at the domain layer for two purposes:

1. **Domain Entity Definition**: All core domain objects inherit from `pydantic.BaseModel`. Validators enforce domain invariants at construction time (e.g., a `Concept` cannot have an empty `name` field; a `MasteryScore` cannot exceed `1.0`). This eliminates an entire class of defensive checks scattered throughout the codebase.
2. **Structured LLM Output Parsing**: Pydantic models serve as the schema for structured output extraction from LLM responses via LangChain's `.with_structured_output()`. The LLM is constrained to return JSON conforming to the Pydantic schema, and the result is immediately a typed domain object.

**Relevance to decoupling:** Because Pydantic is a pure-Python data validation library with no framework entanglement, its use in the domain layer does not violate the Dependency Rule. It is a dependency on a *utility*, not on an *infrastructure framework*.

### 4.3 LLM Abstraction: `LangChain Core`

**Rationale:** `langchain-core` provides the abstract interfaces (`BaseLanguageModel`, `BaseChatModel`, `BaseRetriever`, `Runnable`) that decouple application code from any specific LLM provider. The concrete provider packages (`langchain-openai`, `langchain-anthropic`, `langchain-google-genai`) implement these interfaces and live exclusively in `infrastructure/`.

The key abstraction is the `Runnable` protocol, which defines a uniform `.invoke()` / `.stream()` / `.batch()` interface. A LangChain Expression Language (LCEL) chain is itself a `Runnable`, meaning composite chains can be constructed in infrastructure and injected into application-layer ports as opaque callables.

**Relevance to decoupling:** A use case that holds a reference to a `TutorPort` (which wraps a `Runnable` internally) can be tested by injecting a mock `Runnable` that returns a hardcoded response. Switching from GPT-4o to Claude Sonnet 4.5 in production requires changing one line in the composition root — zero changes to use cases or domain logic.

### 4.4 Agent Orchestration: `LangGraph`

**Rationale:** The mentorship flow is inherently cyclic and stateful: assess -> diagnose -> instruct -> re-assess -> (pass threshold?) -> end or loop. This is not representable as a linear chain. LangGraph models this as a directed graph (a `StateGraph`) where:

- **Nodes** are Python functions or callables that receive the current graph state and return a state delta.
- **Edges** are conditional transitions determined by domain logic evaluated on the current state.
- **State** is a typed Pydantic or `TypedDict` schema that accumulates the session's history, current concept, gap analysis, and mastery scores.

LangGraph's `StateGraph` is instantiated and compiled entirely within `infrastructure/`. The node functions are thin wrappers that call use cases from `application/`. The graph itself has no domain logic — it is a routing mechanism.

**Relevance to decoupling:** LangGraph is infrastructure. If the orchestration framework were replaced with a custom state machine or a different library, the use cases being called by each node remain unchanged. The graph is a deployment detail, not a business logic detail.

### 4.5 API Exposure: `FastAPI`

**Rationale:** FastAPI provides asynchronous HTTP routing, automatic OpenAPI schema generation, and a dependency injection system (`Depends`) that integrates cleanly with the composition root pattern.

Route handlers in `adapters/` are kept thin: they parse the request, call the appropriate use case via an injected dependency, and serialize the response. No business logic resides in route handlers.

FastAPI's `Depends` mechanism serves as the runtime wiring point: the composition root registers concrete port implementations as FastAPI dependencies, and they are injected into use cases at request time.

**Relevance to decoupling:** FastAPI is imported only in `infrastructure/` and `adapters/`. Replacing it with a different ASGI framework (e.g., Starlette directly, or a gRPC server) would require rewriting the adapter layer only — the application and domain layers are unaffected.

### 4.6 Vector Database (Optional — RAG): `Qdrant`

**Rationale:** If the system incorporates a Retrieval-Augmented Generation (RAG) pipeline to ground instructional content in a curated knowledge base, Qdrant is the recommended vector store. It provides a self-hostable, Docker-deployable instance for development and a cloud-managed option for production, with a Python client (`qdrant-client`) that supports both.

The integration point is the `ContentRepositoryPort` interface defined in `domain/`. The `QdrantContentRepositoryAdapter` in `infrastructure/` implements this port. The application layer calls `content_repository.retrieve(query, top_k)` and receives typed domain objects — it has no awareness that a cosine similarity search over dense vector embeddings was performed.

**Relevance to decoupling:** The vector store is substitutable. Replacing Qdrant with `pgvector`, Weaviate, or an in-memory index for testing requires implementing the `ContentRepositoryPort` interface on the replacement class — no other code changes.

---

## 5. Agent Orchestration Design (State Graph Approach)

### 5.1 Graph State Schema

The LangGraph `StateGraph` is parameterized by a typed state schema. All fields are immutable across a given node execution; nodes return deltas that are merged into the state by the graph runtime.

```python
# src/infrastructure/graph/state.py
from typing import Annotated
from pydantic import BaseModel
from langgraph.graph.message import add_messages

class TutorGraphState(BaseModel):
    session_id: str
    learner_id: str
    target_concept: str
    assessment_history: list[AssessmentResult] = []
    current_gap: KnowledgeGap | None = None
    mastery_score: MasteryScore = MasteryScore(value=0.0)
    messages: Annotated[list, add_messages] = []
    iteration_count: int = 0
```

### 5.2 Node Definitions

Each node is a function that accepts `TutorGraphState` and returns a partial state update. Nodes do not contain business logic — they delegate to application-layer use cases.

| Node | Responsibility | Calls |
|---|---|---|
| `assess_learner` | Sends an assessment prompt and parses the learner's response into `AssessmentResult` | `EvaluateAssessmentUseCase` |
| `diagnose_gap` | Analyzes `assessment_history` to identify the most critical `KnowledgeGap` | `GapAnalysisService` (domain) |
| `generate_explanation` | Produces targeted instructional content for the identified gap | `GenerateExplanationUseCase` |
| `check_mastery` | Evaluates whether `mastery_score` meets the threshold | Domain logic on `MasteryScore` |
| `end_session` | Persists the completed session and emits a summary | `CompleteSessionUseCase` |

### 5.3 Conditional Edge Logic

State transitions are governed by a conditional edge function evaluated after `check_mastery`. This is the only location in the graph where routing logic is expressed:

```python
# src/infrastructure/graph/edges.py
from src.domain.entities import MASTERY_THRESHOLD

def route_after_mastery_check(state: TutorGraphState) -> str:
    if state.mastery_score.value >= MASTERY_THRESHOLD:
        return "end_session"
    if state.iteration_count >= MAX_ITERATIONS:
        return "end_session"
    return "generate_explanation"
```

`MASTERY_THRESHOLD` is a domain constant. The edge function imports it from the domain layer — this is a legitimate inward dependency. The edge function does not import from `infrastructure/`.

### 5.4 Graph Compilation

The graph is compiled once at application startup within the composition root:

```python
# src/infrastructure/graph/builder.py
from langgraph.graph import StateGraph, END

def build_tutor_graph(use_cases: UseCaseContainer) -> CompiledGraph:
    builder = StateGraph(TutorGraphState)

    builder.add_node("assess_learner",       use_cases.evaluate_assessment.as_node())
    builder.add_node("diagnose_gap",         use_cases.gap_analysis.as_node())
    builder.add_node("generate_explanation", use_cases.generate_explanation.as_node())
    builder.add_node("check_mastery",        use_cases.check_mastery.as_node())
    builder.add_node("end_session",          use_cases.complete_session.as_node())

    builder.set_entry_point("assess_learner")
    builder.add_edge("assess_learner",       "diagnose_gap")
    builder.add_edge("diagnose_gap",         "generate_explanation")
    builder.add_edge("generate_explanation", "check_mastery")
    builder.add_conditional_edges("check_mastery", route_after_mastery_check)
    builder.add_edge("end_session", END)

    return builder.compile()
```

The `UseCaseContainer` is a plain dataclass populated at the composition root by injecting concrete port implementations. The graph has no direct dependency on any LLM client — it receives pre-wired use cases as callables.

### 5.5 Testability of the Graph

Because the graph nodes delegate to use cases, and use cases accept injected ports, the entire graph is testable with fake implementations:

- **Unit tests** replace all ports with synchronous fakes returning hardcoded domain objects. The graph can be exercised end-to-end without a network call.
- **Integration tests** wire the graph with real LLM clients (or a locally running Ollama instance) and verify that the full session lifecycle completes correctly and produces a valid `MasteryScore`.

This separation ensures that the graph's routing logic (which node fires next) can be verified independently of the LLM's response quality.

---

## 6. Module Architecture Levels & Naming Conventions

> **Note for tooling:** This section is the authoritative reference for `/check-arch` and `/new-feature`. All architecture validation and scaffolding decisions must be derived from the definitions below.

### 6.1 Module Registry

The following table defines every domain module in the system, its **architecture level**, and the layers it spans. The architecture level determines the mandatory folder structure validated during the `/check-arch` gate.

| Module | Architecture Level | Description |
|---|---|---|
| `session` | **Full** | Core learning session lifecycle. Spans all 4 layers. |
| `assessment` | **Full** | Learner evaluation: question generation, answer parsing, scoring. |
| `concept` | **Full** | Knowledge domain modeling: concept graph, prerequisites, difficulty. |
| `gap_analysis` | **Standard** | Derives `KnowledgeGap` from `AssessmentResult` history. Domain + Application only. |
| `mastery` | **Standard** | Computes and tracks `MasteryScore` over time. Domain + Application only. |
| `explanation` | **Standard** | Generates targeted instructional content for a given gap. |
| `auth` | **Light** | API key validation and learner identity. Infrastructure + Adapters only. |
| `config` | **Light** | Environment configuration and feature flags. Infrastructure only. |

---

### 6.2 Architecture Level Definitions

#### Level: Full

**When to use:** The feature introduces new domain entities, requires a dedicated port (interface), has a concrete infrastructure implementation, and is exposed via the HTTP API.

**Mandatory folder structure:**

```
src/
├── domain/
│   └── {module}/
│       ├── __init__.py
│       ├── entities.py          # Pydantic domain models
│       ├── ports.py             # ABCs (interfaces)
│       └── services.py          # Domain services (pure logic)
├── application/
│   └── {module}/
│       ├── __init__.py
│       ├── use_cases.py         # Use case classes with execute()
│       └── dtos.py              # Application-layer DTOs
├── adapters/
│   └── {module}/
│       ├── __init__.py
│       ├── controllers.py       # FastAPI route handlers (thin)
│       └── parsers.py           # LLM output -> domain object parsers
└── infrastructure/
    └── {module}/
        ├── __init__.py
        └── adapters.py          # Concrete port implementations
```

**Dependency rules:**
- `domain/{module}/entities.py` -> imports: `pydantic`, stdlib only.
- `domain/{module}/ports.py` -> imports: `abc`, `domain/{module}/entities.py` only.
- `application/{module}/use_cases.py` -> imports: `domain/{module}/ports.py`, `domain/{module}/entities.py` only.
- `adapters/{module}/controllers.py` -> imports: `application/{module}/use_cases.py`, `application/{module}/dtos.py` only. **Never imports from `infrastructure/`.**
- `infrastructure/{module}/adapters.py` -> imports: `domain/{module}/ports.py` (to implement), any third-party library.

---

#### Level: Standard

**When to use:** The feature introduces domain logic and a use case but does not require a dedicated HTTP endpoint or a new infrastructure adapter. It reuses the existing `TutorPort` or other shared ports.

**Mandatory folder structure:**

```
src/
├── domain/
│   └── {module}/
│       ├── __init__.py
│       ├── entities.py
│       └── services.py
└── application/
    └── {module}/
        ├── __init__.py
        ├── use_cases.py
        └── dtos.py
```

**Dependency rules:**
- `domain/{module}/` -> same constraints as Full level.
- `application/{module}/use_cases.py` -> may reference ports from *other* modules (e.g., `domain/session/ports.py`) but never from `infrastructure/`.
- No `adapters/{module}/` or `infrastructure/{module}/` directories are created.

---

#### Level: Light

**When to use:** The feature has no domain logic. It is a cross-cutting concern (e.g., authentication middleware, configuration loading) that lives entirely in the infrastructure or adapter boundary.

**Mandatory folder structure:**

```
src/
└── infrastructure/
    └── {module}/
        ├── __init__.py
        └── {module}.py
```

Or, if it requires an HTTP adapter:

```
src/
├── adapters/
│   └── {module}/
│       ├── __init__.py
│       └── middleware.py
└── infrastructure/
    └── {module}/
        ├── __init__.py
        └── {module}.py
```

**Dependency rules:**
- No files in `domain/` or `application/` for this module.
- `infrastructure/{module}/` may import any third-party library.
- `adapters/{module}/` may import from `infrastructure/{module}/` (exception to the general adapter rule, because there is no application layer to go through).

---

### 6.3 Naming Conventions

All naming follows Python conventions (`snake_case` for files and variables, `PascalCase` for classes).

#### Files

| Artifact | Pattern | Example |
|---|---|---|
| Domain entity | `entities.py` inside module folder | `domain/session/entities.py` |
| Domain port (interface) | `ports.py` inside module folder | `domain/assessment/ports.py` |
| Domain service | `services.py` inside module folder | `domain/gap_analysis/services.py` |
| Use case | `use_cases.py` inside module folder | `application/concept/use_cases.py` |
| Application DTO | `dtos.py` inside module folder | `application/session/dtos.py` |
| Controller (adapter) | `controllers.py` inside module folder | `adapters/session/controllers.py` |
| LLM parser | `parsers.py` inside module folder | `adapters/assessment/parsers.py` |
| Infrastructure adapter | `adapters.py` inside module folder | `infrastructure/concept/adapters.py` |

#### Classes

| Artifact | Pattern | Example |
|---|---|---|
| Domain entity | `{Concept}` | `LearningSession`, `KnowledgeGap` |
| Domain port | `{Concept}Port` | `TutorPort`, `ContentRepositoryPort` |
| Domain service | `{Concept}Service` | `GapAnalysisService` |
| Use case | `{Verb}{Concept}UseCase` | `EvaluateAssessmentUseCase`, `StartSessionUseCase` |
| Application DTO (input) | `{Concept}InputDTO` | `StartSessionInputDTO` |
| Application DTO (output) | `{Concept}OutputDTO` | `AssessmentResultOutputDTO` |
| Infrastructure adapter | `{Provider}{Concept}Adapter` | `LangChainOpenAITutorAdapter`, `QdrantContentRepositoryAdapter` |

#### Test Files

| Test type | Pattern | Location |
|---|---|---|
| Unit test for domain | `test_{module}_{artifact}.py` | `tests/unit/domain/{module}/` |
| Unit test for use case | `test_{module}_{use_case}.py` | `tests/unit/application/{module}/` |
| Integration test | `test_{module}_integration.py` | `tests/integration/{module}/` |

Example: a unit test for `EvaluateAssessmentUseCase` lives at:
`tests/unit/application/assessment/test_assessment_evaluate_assessment_use_case.py`

---

### 6.4 Import Boundary Enforcement

The following import paths are **always forbidden**, regardless of module or level:

| From layer | Forbidden import target | Reason |
|---|---|---|
| `domain/` | `application/`, `adapters/`, `infrastructure/`, any third-party framework | Violates Dependency Rule |
| `application/` | `adapters/`, `infrastructure/`, any concrete class from infrastructure | Violates Dependency Rule |
| `adapters/` | `infrastructure/` (except composition root) | Bypasses application boundary |
| `tests/unit/` | Any class from `infrastructure/` | Unit tests must not require live services |

Enforcement is implemented via `import-linter` rules defined in `pyproject.toml`. CI fails on any import boundary violation.
