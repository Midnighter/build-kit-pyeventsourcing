---
name: build-state-change
description: Implements a pyeventsourcing state-change slice (Slice + DcbApplication, FastAPI route, pytest tests) from a slice.json definition
---

# Build State Change Slice

> Before doing anything else, read the slice definition from `.build-kit/.slices/{Context}/{SliceName}/slice.json` (both segments in their disk form — i.e. `snake_case({Context})/snake_case({SliceName})`). This file is the **source of truth** for all fields, events, and metadata. Never invent fields not defined there.

Project-wide conventions (tooling, pre-commit, test layout) live in `CLAUDE.md`. Consult it for anything not specific to building a slice.

---

## What a State Change Slice is

A state-change slice processes a command using event sourcing. It:
1. Loads the current DCB projection state by replaying the events selected by the slice's consistency boundary
2. Validates the command against that state
3. Emits new events if valid, raises if not

The slice is expressed as an `eventsourcing.pydantic.Slice` subclass and invoked via a `DcbApplication`.

---

## Step 1 — Read the slice.json

From the slice definition, extract:
- **sliceName** — the slice title (becomes the `Slice` subclass name and the command method name)
- **context** — the bounded context (used to find `src/{context}/events.py`)
- **commands[]** — list of commands with their data fields
- **events[]** — list of events emitted by each command
- **specifications[]** — test scenarios. If empty, still write at least one happy-path test and one invariant-violation test (e.g. "already processed").

### Placeholder grammar

Every placeholder in the templates below is **PascalCase**. There is only one form; derive it once from `sliceName` and reuse it verbatim.

| Placeholder | Derived from | Example |
|-------------|--------------|---------|
| `{SliceName}` | `sliceName` in PascalCase | `AdminCancelLicense` |
| `{EventName}` | event type in PascalCase | `LicenceCancelled` |
| `{Context}` | bounded context in PascalCase | `Backoffice` |

Filesystem paths, Python module names, method names, and route prefixes are **derived from the PascalCase placeholder at code-generation time**, not carried as separate placeholders:

- Python module / package path → lowercase PascalCase split on word boundaries, joined with `_` (e.g. `AdminCancelLicense` → `admin_cancel_license`).
- Command method name → same as the Python module (snake_case).
- Route prefix → same rule but joined with `-` (e.g. `AdminCancelLicense` → `admin-cancel-license`).

Apply these transforms mechanically; do not introduce new placeholder tokens.

---

## Step 2 — Ensure the shared events module exists

Each context has one `src/snake_case({Context})/events.py` module holding every domain event for that context — events are shared across slices in the same context.

File location: `src/snake_case({Context})/events.py`.

### Event pattern

```python
from eventsourcing.pydantic import Decision


class {EventName}(Decision):
    """{one-line description of what this event means}."""

    field1: str
    field2: int
    # data fields from slice.json — use snake_case even if slice.json uses camelCase
```

> The template omits the copyright header and module docstring for brevity. Every real file needs them — commit the result and let pre-commit surface anything you missed (see `CLAUDE.md`).

Add each new event type to this module. Do NOT remove existing ones.

---

## Step 3 — Create `slice.py`

File: `src/snake_case({Context})/snake_case({SliceName})/slice.py`

### Choose the consistency boundary tags first

Before you write the slice, decide which **entity** its invariant guards. In DCB, `Selector.tags` scopes the replay: the app only rebuilds decision state from events whose tags **intersect** the selector's tags. Empty `tags=[]` means "every event of this type, everywhere" — a global boundary. That is almost never what you want.

Pick the tag values from the command arguments — never invent them, never derive them from wall-clock or generated data:

| Invariant scope | `tags=` should be |
|-----------------|-------------------|
| "This user can't do X twice" | `[f"user:{user_id}"]` |
| "This licence can't be cancelled twice" | `[f"licence:{licence_id}"]` |
| "At most N members per organisation" | `[f"orga:{orga_id}"]` |
| Two entities together (rare) | `[f"user:{user_id}", f"orga:{orga_id}"]` |
| **Truly global** (a system-wide invariant, e.g. singleton config) | `[]` — and justify it in the docstring |

The same tags must be attached at **emission time** via `trigger_event(..., tags=...)` — see the **Selector tags ⊆ trigger tags** rule in `CLAUDE.md`. Get this wrong and the invariant silently breaks.

### Full structure

```python
from eventsourcing.domain import event
from eventsourcing.pydantic import DcbApplication, Selector, Slice

from snake_case({Context}).events import {EventName}


class {SliceName}Slice(Slice):
    """DCB slice that processes the {SliceName} command."""

    def __init__(self, field1: str, field2: int) -> None:
        # Command arguments live on self — `execute()` takes no args because
        # `DcbApplication.do(slice_instance)` calls `.execute()` with no arguments.
        self.processed = False
        self._field1 = field1
        self._field2 = field2

    def _tags(self) -> list[str]:
        # Consistency boundary keyed by the entity this command mutates.
        # Replace `field1` with whatever id from slice.json identifies the entity.
        return [f"{{entity_kind}}:{self._field1}"]

    def consistency_boundary(self) -> Selector:
        """Return the selector that defines this slice's consistency boundary."""
        # Selector.types is a **Sequence**, not a set. Use a list literal.
        # tags MUST match the tags used at `trigger_event` — see `_tags()`.
        return Selector(types=[{EventName}], tags=self._tags())

    @event({EventName})
    def _(self) -> None:
        # Project the event to attributes used during validation.
        self.processed = True

    def execute(self) -> None:
        """Validate and emit a {EventName} event."""
        if self.processed:
            msg = "already_processed"
            raise ValueError(msg)

        self.trigger_event(
            {EventName},
            self._tags(),
            field1=self._field1,
            field2=self._field2,
        )


class {SliceName}App(DcbApplication):
    """Application that exposes the {SliceName} command."""

    def snake_case({SliceName})(self, field1: str, field2: int) -> None:
        """{One-line description of the command}."""
        # `do()` takes an **instance** of the slice, not the class.
        # It internally calls `slice.execute()` — do NOT call `.execute()` here.
        self.do(
            {SliceName}Slice(field1=field1, field2=field2),
        )
```

Notes on the template:

- `trigger_event`'s second positional argument is the tag sequence — it is positional-only, so pass `self._tags()` before the keyword event fields.
- If the invariant genuinely is global, drop `_tags()` and return `Selector(types=[{EventName}], tags=[])`. Add a one-line docstring on `consistency_boundary` explaining why.

### State complexity guide

| Scenario | Decision attributes |
|----------|---------------------|
| Simple create-once (per entity) | `self.created = False` |
| Idempotency across all entities (rare) | `self.processed_user_ids: set[str] = set()` |
| Count validation (per entity) | `self.count = 0`, `self.limit = N` |
| No validation needed | (only the command args on `self`) |

For the common per-entity case, tag scoping does the heavy lifting — you only need a single `bool` on `self` because the replayed events are already filtered to that entity.

---

## Step 4 — Create `routes.py`

File: `src/snake_case({Context})/snake_case({SliceName})/routes.py`

```python
from datetime import date  # runtime import — Pydantic model field type
from functools import lru_cache
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel

from snake_case({Context}).snake_case({SliceName}).slice import {SliceName}App

router = APIRouter(
    prefix="/kebab-case({SliceName})",
    tags=["snake_case({SliceName})"],
)


@lru_cache(maxsize=1)
def get_snake_case({SliceName})_app() -> {SliceName}App:
    """Return a shared {SliceName}App instance."""
    return {SliceName}App()


class {SliceName}Request(BaseModel):
    """Request body for the {SliceName} command."""

    field1: str
    field2: int


@router.post("/", status_code=status.HTTP_201_CREATED)
async def snake_case({SliceName})(
    body: {SliceName}Request,
    app: Annotated[{SliceName}App, Depends(get_snake_case({SliceName})_app)],
) -> dict[str, str]:
    """{One-line description of the endpoint}."""
    try:
        app.snake_case({SliceName})(**body.model_dump())
    except ValueError as exc:
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            detail=str(exc),
        ) from exc
    return {}
```

### Error mapping

| Exception | HTTP status |
|-----------|-------------|
| `ValueError` (validation failure inside `execute`) | `422 Unprocessable Entity` |
| Domain conflict class (e.g. duplicate) | `409 Conflict` |
| Anything else | let FastAPI return `500` |

Raise domain errors from `execute` (`raise ValueError("already_processed")`) and translate them in the route handler.

---

## Step 5 — Acceptance tests (slice-level, GWT)

File: `tests/acceptance/snake_case({Context})/snake_case({SliceName})/test_snake_case({SliceName}).py`

```python
import pytest
from eventsourcing.dcb.gwt import given
from eventsourcing.domain import TaggedEvent

from snake_case({Context}).snake_case({SliceName}).slice import {SliceName}Slice
from snake_case({Context}).events import {EventName}

# Reuse the same values for the entity id: the tag on `given()` events must
# fall inside the slice's consistency boundary or `when()` will not see them.
_FIELD1 = "value"
_TAGS = [f"{{entity_kind}}:{_FIELD1}"]


def _slice() -> {SliceName}Slice:
    return {SliceName}Slice(field1=_FIELD1, field2=1)


def test_snake_case({SliceName})_emits_snake_case({EventName})() -> None:
    """Happy path: the command emits {EventName} tagged for its entity."""
    given().when(_slice()).then(
        TaggedEvent(
            decision={EventName}(field1=_FIELD1, field2=1),
            tags=_TAGS,
        ),
    )


def test_snake_case({SliceName})_raises_when_already_processed() -> None:
    """Invariant: cannot process the same entity twice."""
    prior = TaggedEvent(
        decision={EventName}(field1=_FIELD1, field2=1),
        tags=_TAGS,
    )
    with pytest.raises(ValueError, match="already_processed"):
        given(prior).when(_slice())
```

### GWT API notes

- `then()` compares `TaggedEvent` **instances**, not classes. Construct the expected event with the same fields **and the same tags** the slice would emit.
- Cross-entity isolation can't be tested here (`CLAUDE.md` explains why GWT rejects out-of-boundary histories). Prove it in the **integration suite**: post two commands with different entity ids and assert both succeed.

---

## Step 6 — Integration tests (API-level, TestClient)

File: `tests/integration/snake_case({Context})/test_snake_case({SliceName}).py`

These prove the FastAPI route wires the slice correctly and returns the right status codes and bodies. They belong in `tests/integration/` — a separate hatch env from acceptance tests.

```python
import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from snake_case({Context}).snake_case({SliceName}).routes import (
    get_snake_case({SliceName})_app,
    router,
)
from snake_case({Context}).snake_case({SliceName}).slice import {SliceName}App


@pytest.fixture
def client() -> TestClient:
    """Return a TestClient with a fresh in-memory app per test."""
    app = FastAPI()
    app.include_router(router)
    fresh_app = {SliceName}App()
    # Override the lru_cache'd factory so state does not leak across tests.
    app.dependency_overrides[get_snake_case({SliceName})_app] = lambda: fresh_app
    return TestClient(app)


def test_snake_case({SliceName})_returns_201(client: TestClient) -> None:
    """A valid request returns HTTP 201."""
    response = client.post(
        "/kebab-case({SliceName})/",
        json={"field1": "value", "field2": 1},
    )
    assert response.status_code == 201


def test_snake_case({SliceName})_twice_returns_422(client: TestClient) -> None:
    """Repeating the command returns HTTP 422 with the domain error message."""
    payload = {"field1": "value", "field2": 1}
    client.post("/kebab-case({SliceName})/", json=payload)
    response = client.post("/kebab-case({SliceName})/", json=payload)
    assert response.status_code == 422
    assert response.json()["detail"] == "already_processed"


def test_snake_case({SliceName})_missing_field_returns_422(client: TestClient) -> None:
    """A request missing a required field returns HTTP 422 (Pydantic validation)."""
    response = client.post("/kebab-case({SliceName})/", json={"field1": "value"})
    assert response.status_code == 422
```

---

## Step 7 — Wire the router into the FastAPI app (if a top-level app exists)

Find the top-level FastAPI application (typically `src/snake_case({Context})/app.py` or `src/main.py`) and add:

```python
from snake_case({Context}).snake_case({SliceName}).routes import router as snake_case({SliceName})_router

app.include_router(snake_case({SliceName})_router)
```

If no top-level app exists yet, skip this — the integration tests build a fresh `FastAPI()` per test and don't depend on it.

---

## Key patterns

- **Decision state lives on `self`.** `__init__` sets defaults; `@event` handlers mutate them; `execute` reads them.
- **Tags scope the boundary; state answers the invariant.** `Selector.tags` narrows the replay to the affected entity; the `bool`/`set`/counter on `self` then answers "has this already happened *here*?". Never rely on state alone with `tags=[]` unless the invariant is genuinely system-wide.
- **Selector tags ⊆ trigger tags.** Every tag your selector asks for must be present on the event you emit; otherwise the next command replays a version of history that doesn't include it.
- **Raise, don't return errors.** Any invalid command must raise; the route translates the exception to an HTTP status.
- **In-memory testing.** With no environment configuration `DcbApplication` uses the in-memory event store — one fresh app per test, no cleanup needed.

---

## Files to create

```
src/snake_case({Context})/
    events.py                                             # add new event `Decision` here (shared across slices)
src/snake_case({Context})/snake_case({SliceName})/
    __init__.py                                           # package marker
    slice.py                                              # Slice subclass + DcbApplication subclass
    routes.py                                             # FastAPI router with POST endpoint
tests/acceptance/snake_case({Context})/snake_case({SliceName})/
    test_snake_case({SliceName}).py                       # slice-level GWT tests (eventsourcing.dcb.gwt)
tests/integration/snake_case({Context})/
    test_snake_case({SliceName}).py                       # API-level tests (fastapi.testclient.TestClient)
```
