# AGENTS.md

> Runtime instruction set for AI coding agents. Load at session start. Every rule is a constraint, not a suggestion.

---

## Priority Order

When any two rules conflict, resolve by this order:

1. Correctness
2. Simplicity
3. Readability
4. Maintainability
5. Extensibility
6. Performance

Prefer explicit, simple implementations. Never optimize prematurely.

---

## Commands

```bash
pip install -e ".[dev]"     # install dependencies
pytest -x -q                # run after every change
ruff check . --fix          # lint and auto-fix (local only)
ruff format .               # format code (local only)
mypy .                      # type check
uvicorn app.main:app --reload  # run locally
```

All four checks must pass before any task is marked complete.  
**CI gates** (no auto-fix): `ruff check .` + `ruff format --check .` + `mypy .` + `pytest -x -q`.

---

## Project Structure

```
src/
  domain/           # Pure business logic — zero framework imports
    entities.py     # Entities and value objects (Pydantic, frozen=True)
    exceptions.py   # Exception hierarchy rooted at DomainError
    interfaces.py   # abc.ABC interfaces for every I/O boundary
  application/      # Use cases — orchestrates domain + interfaces
  infrastructure/   # Concrete I/O: DB implementations, external APIs, queues
  api/              # FastAPI routers only — HTTP parsing + routing, no logic
    deps.py         # All Depends() factories
  config/           # Settings loaded from env/config files, never hardcoded
  worker/           # Background tasks and async workers
tests/
  unit/             # Domain + application; all I/O mocked/faked via interfaces
  integration/      # Real or docker-compose infrastructure; no mocks
docs/               # Architecture decisions, major tradeoffs, diagrams
scripts/            # Helper scripts
```

Constraints:
- Business logic is independent of transport, presentation, and persistence layers.
- Circular imports are forbidden. Modules are cohesive and loosely coupled.
- Dependency direction flows strictly inward: `api → application → domain ← infrastructure`.

---

## Before Writing Code

1. Read the existing code in the target module.
2. Identify the correct architectural layer for the change.
3. Reuse existing patterns before introducing new abstractions.
4. If requirements are ambiguous, ask — do not invent assumptions.

Do not rewrite or refactor existing architecture without explicit justification.

---

## Code Style

- **PEP 8:** Enforced via `ruff`. Fix all warnings, not just errors.
- **Type hints:** Mandatory on every argument and return value, including `-> None`.
- **Naming:** Full descriptive words. `calculate_discount_rate`, not `calc_dr`. Abbreviations only for universally understood terms (`id`, `url`, `api`).
- **Single responsibility:** Functions do exactly one thing. If a description needs "and", split the function.
- **Comments:** Explain *why*, never *what*. Delete any comment that restates the code.
- **No `print()`:** Structured logging only (see Logging).

---

## Architecture Rules

### Domain Layer

- Zero imports from `fastapi`, `sqlalchemy`, or any infrastructure library.
- All entities and value objects use `pydantic.BaseModel` with enforced immutability:

```python
from pydantic import BaseModel, ConfigDict

class Order(BaseModel):
    model_config = ConfigDict(frozen=True)
```

- Input validation happens at the API boundary before reaching domain services.

### Dependency Injection

- Constructor injection (`__init__`) only. No service locators, no module-level singletons.
- Every external I/O dependency (DB session, HTTP client, queue) is injected via an `abc.ABC` interface from `domain/interfaces.py`.
- Concrete implementations are wired in `api/deps.py` via `Depends()`.

### Route Handlers

Exactly three responsibilities: parse HTTP input → call an application service → return HTTP response.

```python
# Correct
@router.post("/orders", status_code=201)
async def create_order(
    payload: CreateOrderRequest,
    current_user: UserContext = Depends(get_current_user),
    service: OrderService = Depends(get_order_service),
) -> OrderResponse:
    return await service.create(payload, actor=current_user)

# Wrong — DB logic belongs in a repository, not a handler
@router.post("/orders")
async def create_order(payload: CreateOrderRequest, db: Session = Depends(get_db)):
    order = Order(**payload.dict())
    db.add(order)
    db.commit()
```

### Repository Pattern

All ORM sessions, raw SQL, and external storage interactions live strictly inside repository classes in `infrastructure/`. Repositories implement interfaces from `domain/interfaces.py`. No layer outside a repository touches storage directly.

---

## Error Handling

```python
# domain/exceptions.py
class DomainError(Exception): ...
class NotFoundError(DomainError): ...
class ConflictError(DomainError): ...
```

- Map domain exceptions → HTTP responses in one global FastAPI exception handler, never per-route.
- Wrap low-level errors with domain context before re-raising:

```python
except DBError as exc:
    raise RepositoryError("failed to persist order") from exc
```

- Never `except: pass`. Never log an exception without re-raising or mapping it.

---

## Testing

- Write test assertions before completing the implementation.
- Unit tests inject fakes or `Mock(spec=Interface)` via the constructor — never `unittest.mock.patch` on internal module scopes.
- Every test covers at least one happy path and one failure path.
- Integration tests run against real/test infrastructure — do not mock the system under test.
- All async tests are marked `@pytest.mark.asyncio`.

```python
import pytest
from unittest.mock import Mock
from src.domain.interfaces import InventoryRepository

@pytest.mark.asyncio
async def test_create_order_raises_when_stock_insufficient() -> None:
    # Arrange
    mock_inventory_repo = Mock(spec=InventoryRepository)
    mock_inventory_repo.get.return_value = Stock(quantity=0)
    service = OrderService(inventory=mock_inventory_repo)

    # Act & Assert
    with pytest.raises(InsufficientStockError):
        await service.create(order_request_factory(), actor=user_context_factory())
```

Coverage below 80% on `src/domain/` and `src/application/` breaks the pipeline.

---

## Logging

```python
import structlog
logger = structlog.get_logger()
logger.info("order_created", operation="create_order", component="OrderService", order_id=order.id)
```

Always include: `operation`, `component`, and relevant entity IDs.  
Never log: passwords, tokens, credit card numbers, or any sensitive PII.

---

## Security

- All external input is untrusted. Validate completely at API boundaries before internal processing.
- Never surface internal stack traces in API responses.
- Authorization is enforced in the application layer. Route handlers extract the auth token and pass a structured, verified `UserContext` to the service — never make auth decisions inside a handler.

```python
# Handler: extract and forward only
current_user: UserContext = Depends(get_current_user)
return await service.create(payload, actor=current_user)

# Service: enforce authorization
def create(self, payload: CreateOrderRequest, actor: UserContext) -> Order:
    if not actor.can_create_orders:
        raise UnauthorizedError("actor lacks order creation permission")
```

---

## Concurrency & State

- Prefer immutable data: `frozen=True` on Pydantic models, `tuple` over `list` for fixed collections.
- Shared mutable state requires an explicit `asyncio.Lock` or `threading.Lock`. Document *why* the state must be shared.
- Never hold a lock during network or disk I/O.
- Every background worker requires lifecycle management, cancellation handling, and structured error propagation.
- Unbounded task spawning is forbidden. Use bounded queues or semaphores to cap concurrency.

---

## Configuration

- All config from environment variables or config files — never hardcoded.
- Use a typed `pydantic_settings.BaseSettings` class with sensible defaults.
- Zero credentials, secrets, API keys, or environment-specific values in source control.

---

## Dependencies

Before adding any package:
1. Can the standard library handle it cleanly?
2. Is the library actively maintained?
3. Does it introduce an oversized transitive dependency tree?

Every dependency requires explicit justification. Prefer fewer, well-chosen libraries.

---

## Production Readiness

Features touching production must evaluate:

- Structured contextual logging
- Timeouts on all downstream HTTP/gRPC calls
- Retry logic with exponential backoff
- Graceful shutdown (drain in-flight requests before exit)
- Backpressure on queues and workers
- Concurrency isolation on shared resources

---

## Pull Requests

Changes must be focused (one concern), incremental (no giant diffs), and free of unrelated refactors.

Every PR description must include: problem solved, implementation choices, design tradeoffs, and testing performed.

---

## Non-Goals

Do not introduce unless explicitly required:

- Premature microservice boundaries
- Speculative abstractions ("future-proofing")
- Distributed orchestration at small scale
- Deep inheritance hierarchies
- Custom framework layers over native FastAPI

Build the simplest architecture that satisfies the requirement. Evolve complexity only when constraints demand it.

---

## Definition of Done

- [ ] `pytest -x -q` — zero failures
- [ ] `ruff check .` — zero lint errors
- [ ] `ruff format --check .` — zero formatting discrepancies
- [ ] `mypy .` — zero type errors
- [ ] New behaviour has tests: happy path + at least one failure path
- [ ] Route handlers contain zero domain, persistence, or authorization logic
- [ ] All storage I/O is encapsulated inside repository classes
- [ ] No hardcoded secrets, credentials, or environment-specific values in source
