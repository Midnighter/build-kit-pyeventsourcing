# On-Demand State View

Steps 3–7 for the **on-demand** approach: the projection is rebuilt from the
event store on every query. Read this only after Step 1 of `SKILL.md` selected
this approach.

The projection is expressed as an `eventsourcing.pydantic.Slice` subclass whose
`execute()` is a no-op — the slice exists purely so its `consistency_boundary()`
selector drives the replay. A `DcbApplication` subclass exposes a query method
that constructs the view, replays it, and returns it.

```text
Request
    │
    ▼
DcbApplication.{query}()  # replays matching events and evolves them as view slice attributes
    │
    ▼
Response model
```

---

## Step 3 — Create `projection.py`

File: `src/snake_case({Context})/snake_case({SliceName})/projection.py`

Pick the consistency boundary tags first — see *Consistency boundary tags* in
`SKILL.md`. For a single-entity read model, `tags=[]` is almost never right.

```python
from eventsourcing.domain import event
from eventsourcing.pydantic import DcbApplication, Selector, Slice

from snake_case({Context}).events import {EventName}


class {SliceName}View(Slice):
    """DCB read-model slice that projects {SliceName} state for a single entity."""

    def __init__(self, entity_id: str) -> None:
        # Query arguments live on self — `execute()` takes no args because
        # `DcbApplication.do(slice_instance)` calls `.execute()` with no arguments.
        self._entity_id = entity_id
        self.found = False
        self.field1 = ""
        self.field2 = 0

    def _tags(self) -> list[str]:
        # Consistency boundary keyed by the entity this view reads.
        return [f"{{entity_kind}}:{self._entity_id}"]

    def consistency_boundary(self) -> Selector:
        """Return the selector that defines this view's consistency boundary."""
        # Selector.types is a **Sequence**, not a set. Use a list literal.
        # tags MUST be a subset of the tags used at `trigger_event` in the
        # emitting slice — see `_tags()`.
        return Selector(types=[{EventName}], tags=self._tags())

    @event({EventName})
    def _(self, field1: str, field2: int) -> None:
        # Project the event onto attributes read by the route.
        self.found = True
        self.field1 = field1
        self.field2 = field2

    def execute(self) -> None:
        """Read-only view: no command to run, no event to emit."""
        # Intentionally empty. The replay driven by `consistency_boundary()`
        # populates the attributes above before `do()` returns.


class {SliceName}App(DcbApplication):
    """Application that exposes the {SliceName} query."""

    def snake_case({SliceName})(self, entity_id: str) -> {SliceName}View:
        """{One-line description of the query}."""
        view = {SliceName}View(entity_id=entity_id)
        # `do()` takes an **instance** of the slice, not the class. It replays
        # events matching `consistency_boundary()` through the `@event`
        # handlers, then calls `execute()` (a no-op here).
        self.do(view)
        return view
```

### Read-model complexity guide

| Scenario | Projection attributes |
|----------|-----------------------|
| Single-entity presence check | `self.found = False` plus the fields to expose |
| Append-only list per entity (e.g. tricks per dog) | `self.items: list[str] = []` |
| Count / aggregate per entity | `self.count = 0` |
| Multi-entity collection view (rare) | `self.entries: dict[str, EntryState] = {}` — and re-scope tags |

For the common per-entity case, tag scoping does the heavy lifting: the replayed
events are already filtered to that entity, so a handful of plain attributes on
`self` are enough.

---

## Step 4 — Create `routes.py`

File: `src/snake_case({Context})/snake_case({SliceName})/routes.py`

```python
from functools import lru_cache
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel

from snake_case({Context}).snake_case({SliceName}).projection import {SliceName}App

router = APIRouter(
    prefix="/kebab-case({SliceName})",
    tags=["snake_case({SliceName})"],
)


@lru_cache(maxsize=1)
def get_snake_case({SliceName})_app() -> {SliceName}App:
    """Return a shared {SliceName}App instance."""
    return {SliceName}App()


class {SliceName}Response(BaseModel):
    """Response body for the {SliceName} query."""

    entity_id: str
    field1: str
    field2: int


@router.get("/{entity_id}", response_model={SliceName}Response)
async def snake_case({SliceName})(
    entity_id: str,
    app: Annotated[{SliceName}App, Depends(get_snake_case({SliceName})_app)],
) -> {SliceName}Response:
    """{One-line description of the endpoint}."""
    view = app.snake_case({SliceName})(entity_id=entity_id)
    if not view.found:
        msg = f"{entity_id} not found"
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=msg,
        )
    return {SliceName}Response(
        entity_id=entity_id,
        field1=view.field1,
        field2=view.field2,
    )
```

Notes on the template:

- **`@lru_cache(maxsize=1)`** persists the app across requests so replay uses a single event store. Integration tests must override it (see Step 6).
- **`response_model` matches the return type.** The view class is a `Slice`, not a `BaseModel`; map its attributes onto a Pydantic response model explicitly.

The FastAPI/Pydantic house rules this template follows (`Annotated[…, Depends(…)]`, `EM101` message variables, runtime imports for Pydantic field types) are in `CLAUDE.md`. Status codes follow the *Error mapping* table in `SKILL.md`.

### Collection endpoints

If the view exposes a list (e.g. "all licences for an organisation"), keep the
same pattern but change the slice's `_tags()` to the parent entity, project into
a `list[…]` on `self`, and return `list[{SliceName}Response]`.

---

## Step 5 — Acceptance tests (slice-level, GWT)

File: `tests/acceptance/snake_case({Context})/snake_case({SliceName})/test_snake_case({SliceName}).py`

Slice-level tests drive the view directly through
`eventsourcing.dcb.gwt.given(...).when(...)` and assert on the view's projected
attributes.

```python
from eventsourcing.dcb.gwt import given
from eventsourcing.domain import TaggedEvent

from snake_case({Context}).events import {EventName}
from snake_case({Context}).snake_case({SliceName}).projection import {SliceName}View

# Reuse the same values for the entity id: the tag on `given()` events must
# fall inside the view's consistency boundary or `when()` will not see them.
_ENTITY_ID = "value"
_TAGS = [f"{{entity_kind}}:{_ENTITY_ID}"]


def _view() -> {SliceName}View:
    return {SliceName}View(entity_id=_ENTITY_ID)


def test_snake_case({SliceName})_projects_snake_case({EventName})() -> None:
    """Happy path: replaying {EventName} populates the view's fields."""
    prior = TaggedEvent(
        decision={EventName}(field1="a", field2=1),
        tags=_TAGS,
    )
    view = _view()
    given(prior).when(view)
    assert view.found is True
    assert view.field1 == "a"
    assert view.field2 == 1


def test_snake_case({SliceName})_reports_not_found_when_no_events() -> None:
    """When no events have been emitted, the view reports the entity as absent."""
    view = _view()
    given().when(view)
    assert view.found is False
```

### GWT API notes

- **Stop at `when()`.** `.then(*TaggedEvent)` compares *emitted* events, which a read-only view has none of — assert on the view's projected attributes instead.
- Cross-entity isolation can't be tested here (`CLAUDE.md` explains why GWT rejects out-of-boundary histories). Prove it in the **integration suite**: query two entities and assert each returns its own state.

---

## Step 6 — Integration tests (API-level, TestClient)

File: `tests/integration/snake_case({Context})/test_snake_case({SliceName}).py`

These prove the FastAPI route wires the projection correctly and returns the
right status codes and bodies. They belong in `tests/integration/` — a separate
hatch env from acceptance tests.

```python
import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from snake_case({Context}).snake_case({SliceName}).projection import {SliceName}App
from snake_case({Context}).snake_case({SliceName}).routes import (
    get_snake_case({SliceName})_app,
    router,
)


@pytest.fixture
def client() -> TestClient:
    """Return a TestClient with a fresh in-memory app per test."""
    app = FastAPI()
    app.include_router(router)
    fresh_app = {SliceName}App()
    # Override the lru_cache'd factory so state does not leak across tests.
    app.dependency_overrides[get_snake_case({SliceName})_app] = lambda: fresh_app
    return TestClient(app)


def test_snake_case({SliceName})_missing_entity_returns_404(client: TestClient) -> None:
    """Querying an entity with no events returns HTTP 404."""
    response = client.get("/kebab-case({SliceName})/does-not-exist")
    assert response.status_code == 404
```

To exercise the happy path in integration you need events in the store; write
those through the emitting state-change slice's app (share the `DcbApplication`
instance) or through a fixture that seeds the event store directly. Cross-entity
isolation ("another entity's events do not leak into this one's view") belongs
here, since acceptance-level GWT refuses histories outside the boundary.

---

## Step 7 — Wire the router into the FastAPI app (if a top-level app exists)

Find the top-level FastAPI application (typically `src/snake_case({Context})/app.py`
or `src/main.py`) and add:

```python
from snake_case({Context}).snake_case({SliceName}).routes import router as snake_case({SliceName})_router

app.include_router(snake_case({SliceName})_router)
```

If no top-level app exists yet, skip this — the integration tests build a fresh
`FastAPI()` per test and don't depend on it.

---

## Key patterns

- **Projection state lives on `self`.** `__init__` sets defaults; `@event` handlers mutate them; `execute()` is a no-op; the route reads them and maps to a response model.
- **Tags scope the boundary; attributes answer the query.** `Selector.tags` narrows the replay to the affected entity; the fields on `self` then reflect that entity's projected state. Never rely on state alone with `tags=[]` unless the read model is genuinely system-wide.
- **In-memory testing.** With no environment configuration `DcbApplication` uses the in-memory event store — one fresh app per test, no cleanup needed.

---

## Files to create

```
src/snake_case({Context})/
    events.py                                             # shared event Decisions (add new types here; do not remove existing ones)
src/snake_case({Context})/snake_case({SliceName})/
    __init__.py                                           # package marker
    projection.py                                         # View Slice + DcbApplication subclass
    routes.py                                             # FastAPI router with GET endpoint
tests/acceptance/snake_case({Context})/snake_case({SliceName})/
    test_snake_case({SliceName}).py                       # slice-level GWT tests (eventsourcing.dcb.gwt)
tests/integration/snake_case({Context})/
    test_snake_case({SliceName}).py                       # API-level tests (fastapi.testclient.TestClient)
```
