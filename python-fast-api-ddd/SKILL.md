---
name: python-fast-api-ddd
description: Use when designing, implementing, reviewing, or testing Python FastAPI backends that follow Domain-Driven Design, Onion/Clean Architecture, repository and use case patterns, SQLAlchemy infrastructure adapters, Pydantic presentation schemas, pytest unit tests, or uv/ruff/mypy/pytest tooling.
---

# Python FastAPI DDD

Use this skill to build or review Python FastAPI services shaped around Domain-Driven Design and Onion Architecture. It combines the upstream architecture, presentation, testing, and tooling skills into one Codex skill.

The source material is based on Takahiro Ikeuchi's `python-fastapi-ddd-skill` and the `dddpy` reference implementation. See [SOURCES.md](references/SOURCES.md) for upstream links and the copied revision.

## Reference Map

Load the narrowest reference that matches the task:

- [ARCHITECTURE.md](references/ARCHITECTURE.md): layer boundaries, directory structure, request flow, DI chain, adding a new aggregate.
- [VALUE_OBJECTS.md](references/VALUE_OBJECTS.md): frozen dataclass value objects, validation, IDs, enums, composite values.
- [ENTITIES.md](references/ENTITIES.md): identity-based equality, encapsulation, state transitions, factories.
- [REPOSITORIES.md](references/REPOSITORIES.md): domain repository interfaces, SQLAlchemy adapters, DTO mapping, FastAPI dependency wiring.
- [USECASES.md](references/USECASES.md): one workflow per use case, `execute()`, factories, domain exception flow.
- [PRESENTATION.md](references/PRESENTATION.md): route handlers, Pydantic request/response schemas, error mapping, OpenAPI responses.
- [TESTING.md](references/TESTING.md): pytest patterns for value objects, entities, and use cases with repository mocks.
- [TOOLING.md](references/TOOLING.md): uv, ruff, mypy, pytest, Makefile, CI, FastAPI lifespan, SQLAlchemy bootstrap.

## Core Workflow

1. Inspect the existing project first: `pyproject.toml`, package layout, tests, dependency policy, and agent instructions.
2. Pick the relevant layer before editing. Keep changes close to that layer and do not leak outer-framework concerns inward.
3. Model business concepts in the Domain layer first: Value Objects for validated values, Entities for identity and state, domain exceptions for business failures, and repository interfaces for persistence contracts.
4. Add UseCases as application workflows. Use one class per use case with one public `execute()` method. Depend on domain repository interfaces, not concrete SQLAlchemy adapters.
5. Put FastAPI, Pydantic, SQLAlchemy sessions, DTOs, and dependency wiring outside the Domain layer.
6. Add focused tests for changed behavior. Prefer pure domain tests without mocks and use case tests with `Mock(spec=RepositoryInterface)`.
7. Run the repository's configured format, lint, type, and test commands before reporting completion.

## Layer Rules

Dependencies point inward:

```text
Presentation -> UseCase -> Domain
Infrastructure -> Domain
Composition/DI -> Presentation, UseCase, Infrastructure
```

- Domain has zero FastAPI, Pydantic, SQLAlchemy, settings, logging, or HTTP imports.
- UseCase coordinates workflows and transactions at the application boundary; it does not know about HTTP requests or ORM models.
- Infrastructure implements domain ports such as repositories and converts between domain objects and persistence DTOs.
- Presentation parses HTTP input, constructs Value Objects, calls UseCases, maps domain exceptions to HTTP responses, and converts Entities to response schemas.
- Composition roots wire concrete dependencies with FastAPI `Depends()` or project-local DI helpers.

## Dependency Policy

Honor the target repository's dependency rules. If FastAPI, SQLAlchemy, Pydantic, pytest, ruff, mypy, or uv are not installed yet, do not add them unless the user asks. You can still create domain/use case structure, tests that use existing tools, or documentation that marks future framework integration points.

## Naming And Structure

Use project conventions when they already exist. For greenfield DDD code, use a clear aggregate-centered layout:

```text
app/
  domain/{aggregate}/
    entities/
    value_objects/
    repositories/
    exceptions/
  usecase/{aggregate}/
  infrastructure/{database_or_adapter}/{aggregate}/
  presentation/api/{aggregate}/
    handlers/
    schemas/
    error_messages/
```

In `src/` layout repositories, put the application package under `src/` and keep tests under `tests/`.

## Implementation Defaults

- Value Objects: `@dataclass(frozen=True)`, validate in `__post_init__`, expose primitive values explicitly.
- Entities: compare by ID, keep state private, expose behavior methods for valid state transitions.
- Repositories: define abstract interfaces in Domain; return Domain objects, not ORM rows.
- DTOs/adapters: convert through `to_entity()` and `from_entity()` or equivalent named methods.
- UseCases: accept Value Objects and domain primitives, return Domain objects or explicit result types.
- Presentation schemas: separate request and response models; use `from_entity()` for responses.
- Errors: raise domain-specific exceptions inside Domain/UseCase and map them to HTTP status codes only in Presentation.
- Tests: assert invariants, transitions, and repository interactions at the lowest meaningful layer.

## Common Mistakes

- Importing FastAPI, Pydantic, SQLAlchemy, or settings into Domain.
- Treating Pydantic models or ORM rows as Entities.
- Putting validation only in request schemas when it represents business rules.
- Injecting concrete repositories into UseCases instead of repository interfaces.
- Creating generic service classes that hide multiple workflows behind one broad method set.
- Testing use cases with unspecced mocks that allow misspelled repository methods.
