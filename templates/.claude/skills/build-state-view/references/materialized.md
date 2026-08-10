# Materialized State View

Steps 3–7 for the **materialized** approach: a background subscription
consumes events as they are recorded and updates a standalone read model. The
FastAPI route only ever reads that model — it never touches the event store.
Read this only after Step 1 of `SKILL.md` selected this approach.

> **The central-application pattern does not apply here.** Command slices and
> on-demand views share one process-wide `{ProjectName}App` (see `CLAUDE.md` →
> *Application wiring*). `ProjectionRunner` instead takes an application
> **class** and constructs its own instance (`application_class(env=env)`), so it
> cannot be handed the shared instance. Keep the runner-owned application as
> written below until that gap is closed separately.

The projection is expressed as an `eventsourcing.projection.Projection`
subclass whose `process_event(envelope, tracking)` mutates a
`TrackingRecorder` (the materialized view). A `ProjectionRunner` wires a
`DcbApplication`, the `Projection`, and the view's recorder class together,
subscribes to the application in a background thread, and keeps the view's
tracked position in sync. The FastAPI app starts the runner in its lifespan
and hands the route a reference to `runner.view`.

The view is a `TrackingRecorder`, and **which concrete recorder backs it is a
deployment choice, not a design choice** — see *Choosing the view's backend*
below. Write the view as an abstract interface plus one implementation per
backend you actually deploy; everything downstream (projection, route, tests)
depends only on the interface.

```text
DcbApplication.do(slice)                  (write side, unchanged)
    │  records TaggedEvent
    ▼
ProjectionRunner                          (background thread)
    │  subscribes via app.application_subscription(topics=...)
    ▼
{SliceName}Projection.process_event(envelope, tracking)
    │  mutates the view and calls view.insert_tracking(tracking) exactly once
    ▼
{SliceName}View (TrackingRecorder)        (the materialized state)
    │                                     backed by POPO or Postgres
    ▼
GET route reads {SliceName}View directly  (no event-store access)
```

---

## Choosing the view's backend

`{SliceName}View` extends `eventsourcing.persistence.TrackingRecorder`, which
is abstract. The library ships two concrete tracking recorders usable with a
DCB application:

| Recorder | Module | Use for |
|----------|--------|---------|
| `POPOTrackingRecorder` | `eventsourcing.popo` | In-memory. The default, and what every test in Steps 5–6 uses. Lost on process exit. |
| `PostgresTrackingRecorder` | `eventsourcing.postgres` | Durable, shared across processes. Production. |

> The library also ships a `SQLiteTrackingRecorder`, but **do not use it here.**
> The DCB write side provides factories only for `eventsourcing.dcb.popo` and
> `eventsourcing.dcb.postgres_tt`, so a `DcbApplication` cannot run on SQLite —
> and a durable view paired with a non-durable event store buys nothing. POPO
> and Postgres are the two supported options.

You do not choose the backend by importing a different class at the call site.
`ProjectionRunner` builds the view through
`InfrastructureFactory.construct(...).tracking_recorder(view_class)`, and the
factory is resolved from the **`PERSISTENCE_MODULE` environment variable**. So:

- The `view_class` you pass to `ProjectionRunner` must subclass the concrete
  recorder matching the configured factory — a `POPOFactory` asserts
  `issubclass(view_class, POPOTrackingRecorder)`, `PostgresFactory` asserts
  `PostgresTrackingRecorder`. Passing the wrong pair fails at startup.
- The environment is **name-scoped to `Projection.name`**. `Environment.get`
  tries `{NAME.upper()}_{KEY}` before the bare `{KEY}`, so
  `upper_snake_case({SliceName})_PERSISTENCE_MODULE` configures *this view* without
  touching the `DcbApplication`'s own persistence. The view and the event store
  can sit on different backends.

Default to **POPO only** unless the slice definition asks for durability. Write
the Postgres implementation only when it is actually deployed — an unused
second implementation is dead code that still has to be maintained. The
abstract `{SliceName}View` interface is what makes adding it later cheap.

### Backend-specific mechanics

The two implementations do not share a locking or persistence idiom — do not
copy one into the other:

| | POPO | Postgres |
|---|---|---|
| Mutual exclusion | `with self._database_lock:` (an `RLock`, POPO-only — it does not exist on the SQL recorders) | a database transaction: `with self.datastore.transaction(commit=True) as curs:` |
| Recording tracking | `self._insert_tracking(tracking)` | `self._insert_tracking(curs, tracking)` — takes the cursor, so the entry and its tracking row commit atomically |
| Duplicate-tracking check | call `self._assert_tracking_uniqueness(tracking)` yourself | enforced by the tracking table's primary key; `_insert_tracking` raises `IntegrityError` |
| Extra tables | none — plain Python containers on `self` | append `CREATE TABLE IF NOT EXISTS ...` to `self.sql_create_statements` in `__init__`; the factory calls `create_table()` when `CREATE_TABLE` is not disabled |

In both cases the point is the same: **the entry and its `Tracking` must be
persisted in one atomic step**, so a crash can't leave the view holding data
whose position was never recorded (or vice versa). That is what `add_entry`
takes a `tracking` argument for.

---

## Step 3 — Create `projection.py`

File: `src/snake_case({ProjectName})/snake_case({Context})/snake_case({SliceName})/projection.py`

Four pieces live here: the view's abstract interface, its concrete
implementation(s), the `Projection` that feeds it, and a small module-level
`create_runner()` factory used by both the app lifespan (Step 7) and tests
(Steps 5–6).

Pick the consistency boundary as described under *Consistency boundary tags*
in `SKILL.md`, but note that here it constrains the `topics` the subscription
cares about, not a `Selector`. A materialized view has no per-entity replay
boundary at query time: `process_event` fires for *every* recorded event whose
type is in `topics`, and the view itself decides how to key its state
(typically by an id carried on the event body, not by DCB tags, since a
background subscription cannot scope itself to "one entity").

```python
from abc import abstractmethod
from collections import defaultdict
from dataclasses import dataclass

from eventsourcing.domain import TaggedEvent
from eventsourcing.persistence import Tracking, TrackingRecorder
from eventsourcing.popo import POPOTrackingRecorder
from eventsourcing.projection import Projection, ProjectionRunner
from eventsourcing.pydantic import Decision, DcbApplication
from eventsourcing.utils import EnvType, get_topic

from snake_case({ProjectName}).snake_case({Context}).events import {EventName}


@dataclass
class {EntryName}:
    """A single entry projected into the {SliceName} view."""

    field1: str
    field2: int


class {SliceName}View(TrackingRecorder):
    """Abstract materialized view for {SliceName}."""

    @abstractmethod
    def get_entries(self, entity_id: str) -> list[{EntryName}] | None:
        """Return the entries for `entity_id`, or None if none were ever recorded."""

    @abstractmethod
    def add_entry(
        self,
        entity_id: str,
        entry: {EntryName},
        tracking: Tracking,
    ) -> None:
        """Append an entry for `entity_id`, atomically with the tracking position."""


class POPO{SliceName}View(POPOTrackingRecorder, {SliceName}View):
    """In-memory {SliceName} view, backed by the POPO tracking recorder."""

    def __init__(self) -> None:
        super().__init__()
        self._entries: dict[str, list[{EntryName}]] = defaultdict(list)

    def get_entries(self, entity_id: str) -> list[{EntryName}] | None:
        """Return the entries for `entity_id`, or None if none were ever recorded."""
        with self._database_lock:
            if entity_id not in self._entries:
                return None
            return list(self._entries[entity_id])

    def add_entry(
        self,
        entity_id: str,
        entry: {EntryName},
        tracking: Tracking,
    ) -> None:
        """Append an entry for `entity_id`, atomically with the tracking position."""
        with self._database_lock:
            self._assert_tracking_uniqueness(tracking)
            self._entries[entity_id].append(entry)
            self._insert_tracking(tracking)


class {SliceName}Projection(Projection[{SliceName}View, TaggedEvent[Decision]]):
    """Projection that maintains the {SliceName} materialized view."""

    name = "snake_case({SliceName})"
    topics = (get_topic({EventName}),)

    def process_event(
        self,
        envelope: TaggedEvent[Decision],
        tracking: Tracking,
    ) -> None:
        """Dispatch on decision type and update the view, or just track the position."""
        match envelope.decision:
            case {EventName}(field1=field1, field2=field2, entity_id=entity_id):
                self.view.add_entry(
                    entity_id,
                    {EntryName}(field1=field1, field2=field2),
                    tracking,
                )
            case _:
                self.view.insert_tracking(tracking)


def create_runner(
    view_class: type[{SliceName}View] = POPO{SliceName}View,
    env: EnvType | None = None,
) -> ProjectionRunner:
    """Construct the {SliceName} projection runner (app + projection + view).

    Defaults to the in-memory view. Pass `view_class` plus a matching `env`
    (see *Choosing the view's backend*) to materialize into Postgres instead.
    """
    return ProjectionRunner(
        application_class=DcbApplication,
        projection_class={SliceName}Projection,
        view_class=view_class,
        env=env,
    )
```

`EnvType` comes from `eventsourcing.utils` (it is just `Mapping[str, str]`).

### Optional — a Postgres-backed implementation

Add this **only if the slice is deployed with a durable view**; otherwise the
POPO implementation above is the whole story. Note how it differs from the POPO
version on every point in the *Backend-specific mechanics* table:

```python
from typing import Any

from eventsourcing.postgres import PostgresDatastore, PostgresTrackingRecorder
from psycopg.sql import SQL, Identifier


class Postgres{SliceName}View(PostgresTrackingRecorder, {SliceName}View):
    """Durable {SliceName} view, backed by the Postgres tracking recorder."""

    def __init__(self, datastore: PostgresDatastore, **kwargs: Any) -> None:
        super().__init__(datastore, **kwargs)
        self.entries_table_name = "snake_case({SliceName})_entries"
        self.check_identifier_length(self.entries_table_name)
        self.sql_create_statements.append(
            SQL(
                "CREATE TABLE IF NOT EXISTS {0}.{1} ("
                "entity_id text, field1 text, field2 bigint)",
            ).format(
                Identifier(self.datastore.schema),
                Identifier(self.entries_table_name),
            ),
        )

    def get_entries(self, entity_id: str) -> list[{EntryName}] | None:
        """Return the entries for `entity_id`, or None if none were ever recorded."""
        with self.datastore.transaction(commit=False) as curs:
            curs.execute(
                SQL("SELECT field1, field2 FROM {0}.{1} WHERE entity_id=%s").format(
                    Identifier(self.datastore.schema),
                    Identifier(self.entries_table_name),
                ),
                (entity_id,),
            )
            rows = curs.fetchall()
        if not rows:
            return None
        return [
            {EntryName}(field1=row["field1"], field2=row["field2"]) for row in rows
        ]

    def add_entry(
        self,
        entity_id: str,
        entry: {EntryName},
        tracking: Tracking,
    ) -> None:
        """Append an entry for `entity_id`, atomically with the tracking position."""
        with self.datastore.transaction(commit=True) as curs:
            curs.execute(
                SQL("INSERT INTO {0}.{1} VALUES (%s, %s, %s)").format(
                    Identifier(self.datastore.schema),
                    Identifier(self.entries_table_name),
                ),
                (entity_id, entry.field1, entry.field2),
            )
            self._insert_tracking(curs, tracking)
```

The `datastore` argument is supplied by `PostgresFactory.tracking_recorder`,
which also passes a `tracking_table_name` derived from `Projection.name` and
calls `create_table()` — so `__init__` must accept and forward `**kwargs`
rather than fixing its own signature.

To run against it, pass both the class and a matching environment. Note that
`ProjectionRunner` hands the same `env` to **both** the `DcbApplication` and
the view's factory, so it needs two `PERSISTENCE_MODULE` entries — the
unprefixed one selects the DCB write-side backend, the prefixed one selects
this view's:

```python
create_runner(
    view_class=Postgres{SliceName}View,
    env={
        # Write side — the DCB event store. Its module is the dcb variant.
        "PERSISTENCE_MODULE": "eventsourcing.dcb.postgres_tt",
        # Read side — this view only, prefixed with the projection name.
        "upper_snake_case({SliceName})_PERSISTENCE_MODULE": "eventsourcing.postgres",
        # Connection settings, shared by both because they are unprefixed.
        "POSTGRES_DBNAME": "...",
        "POSTGRES_HOST": "...",
        "POSTGRES_USER": "...",
        "POSTGRES_PASSWORD": "...",
    },
)
```

Read these from the process environment (`os.environ`) in the app's lifespan —
never hard-code credentials in the module. A durable view over an in-memory
event store is not a useful configuration: on restart the store is empty and
the view is stale forever, so move both sides together or neither.

The repo's `compose.yaml` starts a PostgreSQL matching these settings
(`docker compose up -d`), for manually exercising a durable view. It is
deliberately **not** wired into any test env — the suites stay on POPO.

Both tables are created automatically on first use: the factory derives the
tracking table's name from `Projection.name`
(`snake_case({SliceName})_tracking`) and calls `create_table()`, which also
executes the entries-table statement registered in `__init__`.

### Notes on the template

`CLAUDE.md` covers the `Projection` API rules this template relies on —
`name`, `topics`, matching on `envelope.decision`, the mandatory `case _:`
wildcard, and persisting `tracking` on every branch. Slice-specific points:

- **`Projection.name` is the snake_case slice name**, so it lines up with the
  route and module naming used everywhere else in this skill.
- **`create_runner()` defaults to the POPO view and `env=None`**, which needs
  no configuration at all. Both parameters exist so a deployment can swap in
  `Postgres{SliceName}View` from Step 7's lifespan without touching this
  module; tests keep calling `create_runner()` bare.
- **Keep `{SliceName}View` abstract and depend on it everywhere.** The
  projection is typed `Projection[{SliceName}View, ...]` and the route depends
  on `{SliceName}View`, never on a concrete recorder — that is what lets the
  backend change without touching either.
- **Entity keying is application-level, not DCB-level.** There is no
  per-request consistency boundary here — `add_entry`/`get_entries` key by
  whatever id field the event carries (`entity_id` in the template). Take that
  id straight from the event's `case` pattern; do not invent a tag-derived key.

### Read-model complexity guide

Every collection is keyed by entity id inside the view rather than scoped by a
replay boundary. Shapes below are the POPO implementation; the Postgres
equivalent is a table keyed on `entity_id`, and the abstract interface is
identical either way:

| Scenario | POPO view shape | Postgres equivalent |
|----------|-----------------|---------------------|
| Single-entity presence check | `dict[str, bool]` or `dict[str, {EntryName}]`, `get_entries` returns `None` for "never seen" | one row per entity, `entity_id` primary key |
| Append-only list per entity | `dict[str, list[{EntryName}]]`, as above | many rows per entity, index on `entity_id` |
| Count / aggregate per entity | `dict[str, int]`, increment in `process_event` | `INSERT ... ON CONFLICT (entity_id) DO UPDATE SET n = n + 1` |
| System-wide singleton (rare) | a single mutable attribute guarded by `self._database_lock` | a one-row table; justify it in the docstring, same as an empty-tags selector |

---

## Step 4 — Create `routes.py`

File: `src/snake_case({ProjectName})/snake_case({Context})/snake_case({SliceName})/routes.py`

The route depends on the **view**, not the application — the whole point of
materializing is that a query never touches the event store. The view
instance is created once by the runner in the app's lifespan (Step 7) and
handed to the route through `request.app.state`.

```python
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Request, status
from pydantic import BaseModel

from snake_case({ProjectName}).snake_case({Context}).snake_case({SliceName}).projection import (
    {SliceName}View,
)

router = APIRouter(
    prefix="/kebab-case({SliceName})",
    tags=["snake_case({SliceName})"],
)


def get_snake_case({SliceName})_view(request: Request) -> {SliceName}View:
    """Return the {SliceName} view stored on the app by the projection lifespan."""
    return request.app.state.snake_case({SliceName})_view


class {EntryName}Response(BaseModel):
    """A single entry in the {SliceName} response."""

    field1: str
    field2: int


@router.get("/{entity_id}", response_model=list[{EntryName}Response])
async def snake_case({SliceName})(
    entity_id: str,
    view: Annotated[{SliceName}View, Depends(get_snake_case({SliceName})_view)],
) -> list[{EntryName}Response]:
    """{One-line description of the endpoint}."""
    entries = view.get_entries(entity_id)
    if entries is None:
        msg = f"{entity_id} not found"
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=msg,
        )
    return [
        {EntryName}Response(field1=entry.field1, field2=entry.field2)
        for entry in entries
    ]
```

Notes on the template:

- **No `@lru_cache`'d app factory here.** There is nothing to cache — the
  view is a long-lived object owned by the lifespan-managed `ProjectionRunner`,
  not something a dependency function constructs per call.
- **`request.app.state` is the seam integration tests override.** Step 6
  builds a fresh `FastAPI()`, runs its own `ProjectionRunner` in the test's
  lifespan, and lets the same `request.app.state` attribute flow through —
  no `dependency_overrides` needed for the view itself.
- Status codes follow the *Error mapping* table in `SKILL.md` — staleness
  (a write not yet caught up by the projection) is not distinguished from
  genuine absence; both read as 404. A request arriving before the runner's
  background thread has processed a very recent write is a real race in any
  materialized view — call it out in the endpoint docstring if the slice
  definition's specifications imply low staleness tolerance, but do not add
  speculative retry/backoff logic that the slice.json doesn't ask for.

### Collection endpoints

If the view exposes a list (e.g. "all licences for an organisation"), the
template above already does this — `get_entries` returns `list[...] | None`.
For a single-entity scalar view, change `get_entries`/`add_entry` to
`get_entry`/`set_entry` and return one `{EntryName}Response`, mirroring the
"Single-entity presence check" row in the complexity guide.

---

## Step 5 — Acceptance tests (projection-level)

File: `tests/acceptance/snake_case({Context})/snake_case({SliceName})/test_snake_case({SliceName}).py`

GWT cannot drive a `Projection` (see `CLAUDE.md`), so acceptance tests here
construct the view and projection directly and call `process_event`
themselves — no runner, no background thread, no `ProjectionRunner` involved:

```python
from eventsourcing.domain import TaggedEvent
from eventsourcing.persistence import Tracking

from snake_case({ProjectName}).snake_case({Context}).events import {EventName}
from snake_case({ProjectName}).snake_case({Context}).snake_case({SliceName}).projection import (
    {EntryName},
    {SliceName}Projection,
    POPO{SliceName}View,
)

_ENTITY_ID = "value"


def _projection() -> tuple[{SliceName}Projection, POPO{SliceName}View]:
    view = POPO{SliceName}View()
    return {SliceName}Projection(view=view), view


def test_snake_case({SliceName})_projects_snake_case({EventName})() -> None:
    """Happy path: processing {EventName} adds an entry to the view."""
    projection, view = _projection()
    envelope = TaggedEvent(
        decision={EventName}(entity_id=_ENTITY_ID, field1="a", field2=1),
        tags=[f"{{entity_kind}}:{_ENTITY_ID}"],
    )
    projection.process_event(envelope, Tracking("upstream", 1))
    assert view.get_entries(_ENTITY_ID) == [{EntryName}(field1="a", field2=1)]


def test_snake_case({SliceName})_reports_absent_when_no_events() -> None:
    """An entity with no processed events is absent, not an empty list."""
    _projection_unused, view = _projection()
    assert view.get_entries(_ENTITY_ID) is None
```

### Acceptance-test notes

- **Construct `Tracking(context_name, notification_id)` yourself** — pick any
  `context_name` string (consistent within a test) and strictly increasing
  `notification_id` values, mirroring what `DcbApplicationSubscription` would
  hand the projection in production. Reusing an id raises `IntegrityError`
  (see `CLAUDE.md`).
- **Always test against `POPO{SliceName}View`,** even when the slice also ships
  a Postgres implementation. It needs no configuration, no server, and no
  cleanup. The behaviour under test is the projection's dispatch logic, which
  is backend-independent by construction. If a Postgres implementation exists,
  cover *it* with a separate opt-in test that skips when no database is
  reachable — do not make the whole suite require one.
- **This bypasses `topics` filtering entirely.** Calling `process_event`
  directly means the test is responsible for only sending events the
  projection is meant to handle. A test asserting the projection *ignores* an
  unrelated event type is still valid and useful — send a `TaggedEvent` for a
  type not in the `match`, and assert `view.max_tracking_id(...)` still
  advanced (proving the `_` branch tracked it) while `get_entries` for any
  entity stayed unaffected.

---

## Step 6 — Integration tests (API-level, TestClient + ProjectionRunner)

File: `tests/integration/snake_case({Context})/test_snake_case({SliceName}).py`

These prove the FastAPI route, the lifespan-started `ProjectionRunner`, and
the write path (writing through the same `DcbApplication` the runner
subscribes to) work together end-to-end, including waiting for the
background thread to catch up. Use `runner.wait(notification_id=..., timeout=...)`
to synchronize deterministically — never `time.sleep`.

```python
from collections.abc import AsyncIterator, Iterator
from contextlib import asynccontextmanager

import pytest
from eventsourcing.domain import TaggedEvent
from eventsourcing.projection import ProjectionRunner
from fastapi import FastAPI
from fastapi.testclient import TestClient

from snake_case({ProjectName}).snake_case({Context}).events import {EventName}
from snake_case({ProjectName}).snake_case({Context}).snake_case({SliceName}).projection import create_runner
from snake_case({ProjectName}).snake_case({Context}).snake_case({SliceName}).routes import router


@pytest.fixture
def client() -> Iterator[TestClient]:
    """Return a TestClient whose lifespan runs a fresh ProjectionRunner."""

    @asynccontextmanager
    async def lifespan(app: FastAPI) -> AsyncIterator[None]:
        with create_runner() as runner:
            app.state.snake_case({SliceName})_view = runner.view
            app.state.snake_case({SliceName})_runner = runner
            yield

    app = FastAPI(lifespan=lifespan)
    app.include_router(router)
    with TestClient(app) as test_client:
        yield test_client


@pytest.fixture
def runner(client: TestClient) -> ProjectionRunner:
    """Return the runner started by this app's lifespan."""
    return client.app.state.snake_case({SliceName})_runner


@pytest.fixture
def entity_id() -> str:
    """Return the id shared by the arrangement and the request."""
    return "entity-1"


@pytest.fixture
def prior_thing(runner: ProjectionRunner, entity_id: str) -> {EventName}:
    """Seed the fact this view projects, and wait for it to be projected."""
    decision = {EventName}(entity_id=entity_id, field1="a", field2=1)
    position = runner.app.events.append(
        events=[
            TaggedEvent(decision=decision, tags=[f"{{entity_kind}}:{entity_id}"]),
        ],
    )
    runner.wait(notification_id=position, timeout=5)
    return decision


def test_snake_case({SliceName})_missing_entity_returns_404(client: TestClient) -> None:
    """Querying an entity with no events returns HTTP 404."""
    response = client.get("/kebab-case({SliceName})/does-not-exist")
    assert response.status_code == 404


def test_snake_case({SliceName})_returns_projected_entries(
    client: TestClient, prior_thing: {EventName}, entity_id: str,
) -> None:
    """After a write catches up, the route returns the projected entries."""
    response = client.get(f"/kebab-case({SliceName})/{entity_id}")
    assert response.status_code == 200
    assert response.json() == [
        {"field1": prior_thing.field1, "field2": prior_thing.field2},
    ]


def test_snake_case({SliceName})_isolates_other_entities(
    client: TestClient, runner: ProjectionRunner,
    prior_thing: {EventName}, entity_id: str,
) -> None:
    """Another entity's events do not leak into this entity's view."""
    position = runner.app.events.append(
        events=[
            TaggedEvent(
                decision={EventName}(entity_id="entity-2", field1="b", field2=2),
                tags=["{{entity_kind}}:entity-2"],
            ),
        ],
    )
    runner.wait(notification_id=position, timeout=5)
    response = client.get(f"/kebab-case({SliceName})/{entity_id}")
    assert response.json() == [
        {"field1": prior_thing.field1, "field2": prior_thing.field2},
    ]
```

### Integration-test notes

`CLAUDE.md` covers the mechanics this fixture depends on: the mandatory
`with TestClient(app) as ...` for lifespan-bearing apps, seeding with
`app.events.append(events=[...])`, and `wait()` over `time.sleep`.
Slice-specific points:

- **Seed raw `TaggedEvent`s through the runner's own application.** Unlike the
  other suites, the events must go through `runner.app` — `ProjectionRunner`
  constructs its own `DcbApplication` (see the note at the top of this file),
  so events appended anywhere else are invisible to the subscription.
- **`append()` returns the position `wait()` needs.** That is the whole reason
  the seeding fixture can synchronize: append, then wait on what it returned.
  Never `time.sleep`.
- **Seeding is unconditional, so it repeats freely.** `append()` builds a DCB
  append condition only when `cb`/`after` is given; passing `events=` alone
  imposes none. Two seeds into the same entity's tags therefore both succeed —
  no `advance()`, no `IntegrityError`, and no test-only emitting `Slice` whose
  `consistency_boundary()` you would have to scope by hand.
- **Each seeded fact is its own fixture, declared in the test's signature** —
  never a helper called from the test body. The fixture both appends *and*
  waits, so a test that names it is guaranteed a settled view before its
  first line runs. Fixtures compose, ids get their own fixtures so arrangement
  and URL share one value, and each returns its `Decision` so the test asserts
  against exactly what it arranged.
- **Tags must satisfy Selector tags ⊆ trigger tags.** Seeding under the wrong
  tag is silently invisible rather than an error. The view keys its state by an
  id on the event body, but the tags still have to match what the real emitting
  slice would use, or the projection sees a history the rest of the system does
  not.
- **The lifespan must stash the runner as well as the view.** The route only
  needs `app.state.snake_case({SliceName})_view`, but the seeding fixture needs
  `runner.app` to append through and `runner.wait` to synchronize — hence both
  attributes in the fixture's lifespan.
- **`create_runner()` is called bare here**, so both the `DcbApplication` and
  the view stay in memory — no database, no env vars, no teardown beyond
  exiting the `with` block. The route code exercised is identical to the
  Postgres deployment's, because it depends on the abstract `{SliceName}View`.

---

## Step 7 — Wire the router and the runner lifespan into the FastAPI app

The top-level FastAPI application is `src/snake_case({ProjectName})/main.py`.
Unlike the other approaches, this one *does* touch its `lifespan`, because the
`ProjectionRunner` has to start with the app — combine it with the existing
lifespan using `AsyncExitStack` rather than overwriting it:

```python
from collections.abc import AsyncIterator
from contextlib import AsyncExitStack, asynccontextmanager

from fastapi import FastAPI

from snake_case({ProjectName}).snake_case({Context}).snake_case({SliceName}).projection import create_runner
from snake_case({ProjectName}).snake_case({Context}).snake_case({SliceName}).routes import router as snake_case({SliceName})_router


@asynccontextmanager
async def snake_case({SliceName})_lifespan(app: FastAPI) -> AsyncIterator[None]:
    """Start the {SliceName} projection runner for the lifetime of the app."""
    with create_runner() as runner:
        app.state.snake_case({SliceName})_view = runner.view
        yield


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """Combine every slice's lifespan under one AsyncExitStack."""
    async with AsyncExitStack() as stack:
        await stack.enter_async_context(snake_case({SliceName})_lifespan(app))
        # await stack.enter_async_context(other_slice_lifespan(app))
        yield


app = FastAPI(lifespan=lifespan)
app.include_router(snake_case({SliceName})_router)
```

**This lifespan is the one place the backend is chosen.** The bare
`create_runner()` above materializes into memory, which is the right default —
and correct for a single-process deployment that can afford to rebuild on
restart. If the slice needs a durable view, this is the only line that changes:

```python
    with create_runner(
        view_class=Postgres{SliceName}View,
        env={
            "PERSISTENCE_MODULE": "eventsourcing.dcb.postgres_tt",
            "upper_snake_case({SliceName})_PERSISTENCE_MODULE": "eventsourcing.postgres",
            # POSTGRES_DBNAME / HOST / USER / PASSWORD from os.environ
        },
    ) as runner:
```

`ProjectionRunner` passes this `env` to the `DcbApplication` as well as to the
view's factory, so both sides move to Postgres together — which is what you
want, since a durable view over an in-memory event store goes permanently stale
on the first restart.

Nothing in `routes.py`, the projection, or either test suite is touched — they
all depend on the abstract `{SliceName}View`. Do not add this unless the slice
definition calls for durability.

If no top-level app exists yet, create it with just this one slice's lifespan
entered into the `AsyncExitStack` — the shape stays the same as more slices
are added later, each contributing one more `stack.enter_async_context(...)`
line. If a top-level app already exists with its own `lifespan`, add this
slice's context manager as one more `stack.enter_async_context(...)` call
inside it rather than replacing the function.

---

## Key patterns

- **The view is a `TrackingRecorder`, not a `Slice`.** State lives in a
  standalone recorder object (`POPO{SliceName}View` above); the projection
  mutates it via ordinary methods (`add_entry`, `get_entries`), not `@event`
  handlers.
- **`TrackingRecorder` is abstract; the backend is a deployment choice.**
  `POPOTrackingRecorder` (in-memory) and `PostgresTrackingRecorder` (durable)
  are the two options for a DCB application. Declare an abstract
  `{SliceName}View` and let the projection, route, and tests depend only on
  that, so swapping backends touches one line in the app lifespan.
- **`ProjectionRunner` resolves the recorder from the environment**, not from
  the import. `upper_snake_case({SliceName})_PERSISTENCE_MODULE` selects the factory
  for *this view*; the `view_class` passed in must subclass the matching
  concrete recorder or startup fails an assertion.
- **Entry and tracking must be written atomically** — under
  `self._database_lock` for POPO, inside one `datastore.transaction(commit=True)`
  for Postgres. This is why `add_entry` takes the `Tracking`.
- **The route depends on the view object, not on the application or the
  runner.** The runner (and its write-side `DcbApplication`) exist only to
  keep the view current in the background; the route never imports
  `ProjectionRunner`.
- **Entity keying is application-level, not DCB-level.** A background
  subscription cannot scope itself to one entity, so the view keys its own
  state by an id carried on the event body — there is no per-query replay
  boundary the way there is in the on-demand approach.
- **In-memory testing, whatever the deployment backend.**
  `POPOTrackingRecorder`/`POPOApplicationRecorder` need no environment
  configuration — one fresh `ProjectionRunner` per test (Step 6) or one fresh
  view+projection pair with no runner at all (Step 5), no cleanup needed
  beyond exiting the `with` block. Tests never require a database.

---

## Files to create

```
src/snake_case({ProjectName})/snake_case({Context})/
    events.py                                             # shared event Decisions (add new types here; do not remove existing ones)
src/snake_case({ProjectName})/snake_case({Context})/snake_case({SliceName})/
    __init__.py                                           # package marker
    projection.py                                         # View interface + POPO impl (+ Postgres impl only if deployed) + Projection + create_runner()
    routes.py                                             # FastAPI router reading the view from app.state
tests/acceptance/snake_case({Context})/snake_case({SliceName})/
    test_snake_case({SliceName}).py                       # projection-level tests (direct process_event calls, no runner)
tests/integration/snake_case({Context})/
    test_snake_case({SliceName}).py                       # API-level tests (TestClient + lifespan-started ProjectionRunner)
```
