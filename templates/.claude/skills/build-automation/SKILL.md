---
name: build-automation
description: Implements a pyeventsourcing automation slice (tracking view + Projection that fires a command, pytest tests) from a slice.json definition
---

# Build Automation Slice

> Before doing anything else, read the slice definition from `.build-kit/.slices/<contextSlug>/<sliceFolder>/slice.json`. This file is the **source of truth** for which trigger event drives the automation and what command it fires. Never invent fields not defined there.
>
> **These two path segments are lowercase slugs, not snake_case**, and they slug differently from each other — they are written by the `load-slice` skill, which owns the rules:
>
> - `<contextSlug>` — the context lowercased, spaces to hyphens, non-alphanumeric removed (`"My Ctx"` → `my-ctx`). Falls back to `default` when the slice has no context.
> - `<sliceFolder>` — the slice title lowercased with **all spaces removed** and any `slice:` prefix stripped (`"Cancel Expired License"` → `cancelexpiredlicense`).
>
> So `CancelExpiredLicense` in the `Backoffice` context reads from `.build-kit/.slices/backoffice/cancelexpiredlicense/slice.json`. Do **not** use the `snake_case(...)` form here — that convention applies to the generated Python paths below, not to this directory. If the path you derive is missing, list the context directory rather than guessing.

Project-wide conventions (tooling, pre-commit, test layout) live in `CLAUDE.md`. Consult it for anything not specific to building a slice.

---

## What an Automation Slice is

An automation slice reacts to events and fires a command in response — a
**state-change slice driven by a policy** instead of by an HTTP route.

In this codebase that is a `Projection` running under a runner. The runner
subscribes to a `DcbApplication` via `application_subscription`; for each event
it calls the projection's `process_event`, which maintains a `TrackingRecorder`
view and issues the command. The view holds both the outstanding work and the
position of the last processed event.

```
DcbApplication  ──subscription──▶  Projection.process_event
      ▲                                    │
      │                                    ├──▶ view.add_entry(entry, tracking)
      └────────── command ◀────────────────┘
                     │
                     ▼
             emits new event ──▶ back through the same subscription
                                 └──▶ view.remove_entry(...)
```

The last arrow is the point. Events the automation emits **come back through the
same subscription**, and that is what drains the entry. The view is simultaneously
the automation's input state and its ledger of unfinished work — but only if
something actually reads that ledger back. See Step 4.

---

## Step 1 — Read the slice.json

Extract:
- **sliceName** — the automation's name
- **context** — bounded context
- **processors[]** — the automation element. Its `dependencies` name the inbound read models and the **OUTBOUND `COMMAND`** it fires.
- **commands[]** — the command's data fields
- **events[]** — the events that command emits (these close the loop; see Step 3). Check for a **failure event** among them — a "… failed" outbound event enables the retry loop in Step 4b. Its absence is not a blocker; `drain()` covers recovery either way.
- **specifications[]** — test scenarios. Specs expecting the failure event are describing the automation's retry behaviour, not just the command's validation. If empty, still write at least the trigger-fires-the-command path, the emitted-event-drains-the-entry path, and the `drain()` recovery case.

### Placeholder grammar

Every placeholder in the templates below is **PascalCase**. There is only one form;
derive it once and reuse it verbatim.

| Placeholder | Derived from | Example |
|-------------|--------------|---------|
| `{SliceName}` | `sliceName` in PascalCase | `CancelExpiredLicenses` |
| `{TriggerEventName}` | the inbound event that starts the automation, PascalCase | `LicenseExpired` |
| `{EmittedEventName}` | the event the fired command emits, PascalCase | `LicenseCancelled` |
| `{FailureEventName}` | **optional** — the "… failed" outbound event, PascalCase. Only when the model defines one | `LicenseCancellationFailed` |
| `{EntryName}` | the work-item dataclass, `{SliceName}Entry` unless the model names it | `CancelExpiredLicensesEntry` |
| `{CommandSliceName}` | the `Slice` the automation fires, PascalCase | `CancelLicense` |
| `{Context}` | bounded context in PascalCase | `Backoffice` |
| `{ProjectName}` | `project.name` in `pyproject.toml`, PascalCase | `MyProject` |

`{ProjectName}` is the one placeholder that does **not** come from the slice definition —
read it once from `[project] name` in `pyproject.toml`. It is already fixed for the whole
repository, so `snake_case({ProjectName})` is simply the existing top-level package under
`src/`; confirm it there rather than deriving a name that does not exist on disk.

Filesystem paths, Python module names, and environment-variable prefixes are **derived
from the PascalCase placeholder at code-generation time**, not carried as separate
placeholders:

- Python module / package path → lowercase PascalCase split on word boundaries, joined
  with `_` (e.g. `CancelExpiredLicenses` → `cancel_expired_licenses`).
- Environment-variable prefix → the snake_case form upper-cased, written
  `upper_snake_case({SliceName})` (e.g. `CancelExpiredLicenses` →
  `CANCEL_EXPIRED_LICENSES`). This is what scopes the view's `PERSISTENCE_MODULE` — see
  *Choosing the view's backend* in Step 3.

There is no route-prefix transform: an automation has no `routes.py` (Step 2).

Apply these transforms mechanically; do not introduce new placeholder tokens.

---

## Step 2 — Build the command handler

The command the automation fires is an ordinary state-change slice, and it is a
**separate artifact** — not part of this slice's files-to-create tree. Build it with
**build-state-change**, which creates
`src/snake_case({ProjectName})/snake_case({Context})/events.py` and
`src/snake_case({ProjectName})/snake_case({Context})/snake_case({CommandSliceName})/slice.py`,
then come back here. The automation reuses that `Slice` unchanged; the only difference
is what invokes it.

Two automation-specific constraints that build-state-change will not tell you:

- **Do NOT create a `routes.py`** for the automation itself — it is not driven over HTTP.
  (The command slice may still have one if the model also exposes it to users.)
- **The command slice's boundary must be satisfiable from the trigger event alone.** The
  automation has no request body to fall back on: whatever `consistency_boundary()`
  selects on has to be reconstructible from the fields the trigger carries.

---

## Step 3 — Create `projection.py`

File: `src/snake_case({ProjectName})/snake_case({Context})/snake_case({SliceName})/projection.py`

Three pieces live here: the view (its abstract interface plus a concrete
implementation), the `Projection` that maintains it, and a module-level `create_runner()`
factory used by both the app lifespan and the tests (Step 5).

The templates below are shown one piece at a time. Together they need these imports and
module-level constants:

```python
import contextlib
import logging
import os
from abc import abstractmethod
from collections.abc import Callable
from dataclasses import dataclass, replace

from eventsourcing.domain import (
    TaggedEvent,
    get_metadata_from_context,
    put_metadata_in_context,
)
from eventsourcing.persistence import InfrastructureFactory, Tracking, TrackingRecorder
from eventsourcing.popo import POPOTrackingRecorder
from eventsourcing.projection import BaseProjectionRunner, Projection
from eventsourcing.pydantic import Decision, DcbApplication
from eventsourcing.utils import Environment, EnvType, get_topic

from snake_case({ProjectName}).snake_case({Context}).events import (
    {EmittedEventName},
    {TriggerEventName},
)
from snake_case({ProjectName}).snake_case({Context}).snake_case({CommandSliceName}).slice import (
    {CommandSliceName}Slice,
)

logger = logging.getLogger(__name__)

MAX_ATTEMPTS = 3   # tune per model; see Step 4c
```

Add `{FailureEventName}` to the events import, and `Record{FailureEventName}Slice` to the
slice imports, only when the model defines one (Step 4b). `get_metadata_from_context` is
used by the acceptance tests (Step 6) rather than by `projection.py` itself.

### Choosing the view's backend

`{SliceName}View` extends `eventsourcing.persistence.TrackingRecorder`, which is
abstract. The library ships two concrete tracking recorders usable with a DCB
application:

| Recorder | Module | Use for |
|----------|--------|---------|
| `POPOTrackingRecorder` | `eventsourcing.popo` | In-memory. The default, and what the tests in Steps 6–7 use. Lost on process exit. |
| `PostgresTrackingRecorder` | `eventsourcing.postgres` | Durable, shared across processes. Production. |

> The library also ships a `SQLiteTrackingRecorder`, but **do not use it here.** The DCB
> write side provides factories only for `eventsourcing.dcb.popo` and
> `eventsourcing.dcb.postgres_tt`, so a `DcbApplication` cannot run on SQLite — and a
> durable view paired with a non-durable event store buys nothing.

You do not choose the backend by importing a different class at the call site. The view
is built through `InfrastructureFactory.construct(...).tracking_recorder(view_class)`,
and the factory is resolved from the **`PERSISTENCE_MODULE` environment variable**. So:

- The `view_class` must subclass the concrete recorder matching the configured factory —
  a `POPOFactory` asserts `issubclass(view_class, POPOTrackingRecorder)`,
  `PostgresFactory` asserts `PostgresTrackingRecorder`. The wrong pair fails at startup.
- The environment is **name-scoped to `Projection.name`**. `Environment.get` tries
  `{NAME.upper()}_{KEY}` before the bare `{KEY}`, so
  `upper_snake_case({SliceName})_PERSISTENCE_MODULE` configures *this view* without
  touching the `DcbApplication`'s own persistence. The view and the event store can sit
  on different backends. This is why Step 5 constructs an `Environment` named after the
  projection.

Default to **POPO only** unless the slice definition asks for durability — an unused
second implementation is dead code that still has to be maintained. The abstract
`{SliceName}View` interface is what makes adding it later cheap.

The two implementations do not share a locking or persistence idiom — do not copy one
into the other:

| | POPO | Postgres |
|---|---|---|
| Mutual exclusion | `with self._database_lock:` (an `RLock`, POPO-only — it does not exist on the SQL recorders) | a transaction: `with self.datastore.transaction(commit=True) as curs:` |
| Recording tracking | `self._insert_tracking(tracking)` | `self._insert_tracking(curs, tracking)` — takes the cursor, so the entry and its tracking row commit atomically |
| Duplicate-tracking check | call `self._assert_tracking_uniqueness(tracking)` yourself | enforced by the tracking table's primary key; `_insert_tracking` raises `IntegrityError` |
| Extra tables | none — plain Python containers on `self` | append `CREATE TABLE IF NOT EXISTS ...` to `self.sql_create_statements` in `__init__`; the factory calls `create_table()` when `CREATE_TABLE` is not disabled |

In both cases the point is the same: **the entry and its `Tracking` must be persisted in
one atomic step**, so a crash cannot leave the view holding a work item whose position
was never recorded — or, worse, a recorded position with no work item, which is an
orphaned command that `drain()` can never recover. That is what every mutating method
below takes a `tracking` argument for.

### The view

The automation's view is a **ledger of unfinished work**, not accumulated read-model
state. That shapes the interface: entries are added when the trigger arrives and
**removed** when the emitted event confirms the command landed, so alongside `add_entry`
there is a `remove_entry` — and `get_entries()` takes **no argument**, because `drain()`
(Step 4) needs every outstanding item, not one entity's.

```python
from abc import abstractmethod
from dataclasses import dataclass, replace

from eventsourcing.persistence import Tracking, TrackingRecorder
from eventsourcing.popo import POPOTrackingRecorder


@dataclass
class {EntryName}:
    """One outstanding unit of work: a command that has not yet been confirmed."""

    entity_id: str
    field: str
    attempts: int = 0          # see Step 4c


class {SliceName}View(TrackingRecorder):
    """Abstract ledger of {SliceName} work items."""

    @abstractmethod
    def add_entry(self, entry: {EntryName}, tracking: Tracking) -> None:
        """Record an outstanding entry, atomically with the tracking position."""

    @abstractmethod
    def remove_entry(self, entity_id: str, tracking: Tracking) -> None:
        """Drop the entry for `entity_id`, atomically with the tracking position.

        Must be a no-op when no entry exists — the emitted event may arrive for
        work this process never recorded.
        """

    @abstractmethod
    def get_entries(self) -> list[{EntryName}]:
        """Return copies of every outstanding entry."""

    @abstractmethod
    def count_attempt(self, entity_id: str) -> None:
        """Increment the attempt counter for `entity_id`."""


class POPO{SliceName}View(POPOTrackingRecorder, {SliceName}View):
    """In-memory {SliceName} ledger, backed by the POPO tracking recorder."""

    def __init__(self) -> None:
        super().__init__()
        self._entries: dict[str, {EntryName}] = {}

    def add_entry(self, entry: {EntryName}, tracking: Tracking) -> None:
        """Record an outstanding entry, atomically with the tracking position."""
        with self._database_lock:
            self._assert_tracking_uniqueness(tracking)
            self._entries[entry.entity_id] = entry
            self._insert_tracking(tracking)

    def remove_entry(self, entity_id: str, tracking: Tracking) -> None:
        """Drop the entry for `entity_id`, atomically with the tracking position."""
        with self._database_lock:
            self._assert_tracking_uniqueness(tracking)
            self._entries.pop(entity_id, None)
            self._insert_tracking(tracking)

    def get_entries(self) -> list[{EntryName}]:
        """Return copies of every outstanding entry."""
        with self._database_lock:
            return [replace(entry) for entry in self._entries.values()]

    def count_attempt(self, entity_id: str) -> None:
        """Increment the attempt counter for `entity_id`."""
        with self._database_lock:
            if (entry := self._entries.get(entity_id)) is not None:
                entry.attempts += 1
```

Notes on the template:

- **Keyed by `entity_id`, one entry per entity.** A second trigger for the same entity
  replaces the outstanding entry rather than queueing a duplicate command. If the model
  genuinely allows concurrent work items per entity, key on whatever the model makes
  unique instead — but then `remove_entry` needs that key too, not `entity_id`.
- **`count_attempt` does not take a `tracking`.** It is called from `drain()` and the
  retry path, which are not processing a new event, so there is no position to record.
- **`get_entries()` returns copies** (`replace(entry)`), or callers mutate view state
  through the returned dataclasses.

### topics — list the emitted event too

```python
topics = (get_topic({TriggerEventName}), get_topic({EmittedEventName}))
```

Two topics is the baseline, and both are mandatory. The **emitted** event is what drains
the entry once the command succeeds; omit it and the view grows forever, holding every
licence it ever cancelled.

Add a third topic, `get_topic({FailureEventName})`, **only if** the model defines a
failure event (Step 1). Models without one still work — `drain()` covers recovery either
way (Step 4a).

### The command port

```python
CommandPort = Callable[[{EntryName}], None]


def _no_command(entry: {EntryName}) -> None:
    """Discard the command — the default when no port is injected."""


class {SliceName}Projection(
    Projection[{SliceName}View, TaggedEvent[Decision]],
):
    """..."""

    name = "snake_case({SliceName})"
    topics = (get_topic({TriggerEventName}), get_topic({EmittedEventName}))

    def __init__(
        self,
        view: {SliceName}View,
        command: CommandPort = _no_command,
    ) -> None:
        super().__init__(view=view)
        self._command = command
```

The command is **injected, not reached for**. This is forced by the library:
`ProjectionRunner.__init__` constructs `projection_class(view=view)` and never
passes the application, so a projection has no path to the write side on its own.
Injection is also what makes the projection testable with no app, no runner, and
no background thread.

### process_event

```python
    def process_event(
        self,
        envelope: TaggedEvent[Decision],
        tracking: Tracking,
    ) -> None:
        """..."""
        match envelope.decision:
            case {TriggerEventName}(field=field, entity_id=entity_id):
                entry = {EntryName}(entity_id=entity_id, field=field)
                # Record before commanding: a lingering entry is the observable
                # signal that the command did not land.
                self.view.add_entry(entry, tracking)
                self._fire(entry, _causation_metadata(envelope))
            case {EmittedEventName}(entity_id=entity_id):
                self.view.remove_entry(entity_id, tracking)
            case _:
                self.view.insert_tracking(tracking)
```

**Order matters.** `add_entry(..., tracking)` first, then the command. This gives
at-most-once command delivery and leaves the todo view doubling as the retry
ledger — an entry that never drained is a command that never landed.

Keep the `case _:` wildcard and its `insert_tracking`. `match` is
exhaustive-by-branch, not exhaustive-by-topics, and a branch that skips the
tracking write makes every later `wait()` hang until timeout.

### Firing the command

```python
def _causation_metadata(envelope: TaggedEvent[Decision]) -> dict[str, str]:
    """Derive the metadata naming this envelope as the direct cause."""
    metadata = {}
    with contextlib.suppress(KeyError):
        metadata["correlation_id"] = envelope.metadata["correlation_id"]
    metadata["causation_id"] = str(envelope.uuid)
    return metadata


    def _fire(self, entry: {EntryName}, metadata: dict[str, str]) -> None:
        """Issue the command under the given metadata, swallowing failures."""
        try:
            with put_metadata_in_context(metadata):
                self._command(entry)
        except Exception:
            # An escaping exception permanently kills the runner's processing
            # thread, stalling every later event. Log and leave the entry behind.
            logger.exception("failed to ... for %s", entry.entity_id)
```

`_causation_metadata` is a module-level function, not a method — it needs nothing from
the projection, and keeping it separate is what lets `_fire` take a plain `dict`.

**The guard is mandatory, not defensive style.** An exception escaping
`process_event` kills the runner's processing thread for good: the subscription
stops, `wait()` re-raises the original error, and no subsequent event is ever
processed. A routine domain `ValueError` from the command would take down the
whole automation.

**Metadata propagation** keeps the causal chain intact exactly where no human is
involved. `EventEnvelope.metadata` defaults to `get_metadata_from_context()` and
`TaggedEventMapper` round-trips `metadata`/`uuid` through the store, so wrapping
the call in `put_metadata_in_context` is sufficient — this mirrors what
`EventSourcedProjection.process_event` does natively. `correlation_id` is carried
forward when present (suppressed when absent, so an untagged trigger is not a
crash); `causation_id` is the **trigger event's own uuid**, naming the direct cause.

**`_fire` takes a plain `dict`, not the envelope**, and that is deliberate: recovery
(Step 4a) fires commands with no triggering event at all, and there is no causation to
invent for those. Deriving the metadata at the call site instead of inside `_fire` is
what lets both paths share one method.

---

## Step 4 — Recovery: the orphaned-entry problem

**Guarding the command is not enough.** Read this before assembling the runner —
it is the failure mode most easily missed.

`add_entry` commits the entry *and its tracking position* atomically. A runner
resumes at `gt=tracking_recorder.max_tracking_id(...)`, strictly **after** the last
recorded position. So once `add_entry` succeeds, that trigger event is permanently
consumed. If the command then fails, the guard swallows it, the loop moves on — and
nothing will ever revisit that entry. It sits outstanding forever.

Reversing the order does not help; it just trades a stuck entry for a lost command.
Two mechanisms are needed, and they cover different windows:

### 4a. `drain()` — always implement this

```python
    def drain(self) -> None:
        """Re-issue commands for entries left outstanding by an earlier crash."""
        for entry in self.view.get_entries():
            if entry.attempts > MAX_ATTEMPTS:
                continue
            self.view.count_attempt(entry.entity_id)
            # No triggering envelope exists here, so there is no causation to carry.
            self._fire(entry, {})
```

Call it in `create_runner()` **before** constructing the runner, so it completes
before the subscription starts. This is the baseline recovery path and the **only**
one available when the model has no failure event.

### 4b. Failure events — skip this section unless the model defines one

**This whole section is conditional.** If Step 1 found no failure event among the
command's outbound dependencies, the automation is complete with `drain()` alone — go to
Step 4c, then Step 5, and omit the failure port, the `{FailureEventName}` topic, the
`_retry` branch, and the failure-recording slice entirely.

If `slice.json` *does* list one (e.g. `"License cancellation failed"`), record it and add
it to `topics`. The failure then returns through the same subscription and the retry
becomes ordinary event processing — no timer, no polling, no second thread:

```python
            case {FailureEventName}(entity_id=entity_id):
                self._retry(entity_id, tracking, _causation_metadata(envelope))
```

Like the trigger branch, `_retry` derives its metadata at the call site so the retried
command names the failure event as its cause.

Inject the failure recorder as a **separate optional port** with a no-op default, so the
projection class works unchanged for models that have no such event:

```python
def _no_failure(entry: {EntryName}, reason: str) -> None:
    """Discard the failure — the default for models with no failure event."""
```

Appending the failure event can itself fail, so wrap that call in its own
`try/except` too — otherwise the recovery path becomes a new way to kill the runner.

**The slice that records a failure needs a boundary that never matches.** Recording
a failure is an observation, not a decision: the same entity may fail repeatedly. A
DCB boundary is an append *condition*, so a selector matching prior failures makes
the second recording raise `IntegrityError`.

Give `consistency_boundary()` a selector that cannot match anything in the store: name an
event type this slice never emits, paired with a tag value nothing is ever emitted under.
Both halves matter — a real type with an unused tag still risks matching if another slice
later adopts that tag, and an unused type with a real tag is the collision below.

Note `Selector(types=[], tags=[...])` is **not** "no boundary" — it means "fail if
any event with these tags exists", which is exactly the collision to avoid.

### 4c. Attempt counting

Carry `attempts: int = 0` on the entry and increment it on every fire. Past
`MAX_ATTEMPTS`, stop firing and leave the entry **parked** — still visible in the
view for inspection, rather than retried forever.

The counter earns its keep on the failure-event path (4b), where a permanently failing
command would otherwise loop indefinitely between the failure event and the retry. But
**implement it even without a failure event**: `drain()` checks `attempts` too, so a
crash-orphaned entry whose command fails on every restart stays parked instead of
re-firing on each boot.

---

## Step 5 — Assemble the runner

The stock `ProjectionRunner` cannot inject the command port. It takes *classes* —
`application_class`, `projection_class`, `view_class` — and constructs everything
itself, calling `projection_class(view=view)` with no opportunity to pass anything
else. Since the command port is a constructor argument (Step 3), that closes the door.

`BaseProjectionRunner` is the layer underneath: `ProjectionRunner` subclasses it, doing
the class-to-instance resolution and then delegating. `BaseProjectionRunner` itself takes
an **already-constructed** `projection`, `app`, and `tracking_recorder`, which is exactly
the seam an injected port needs. Both live in `eventsourcing.projection`.

Using it means doing by hand the one thing `ProjectionRunner` was doing for you:
resolving the `InfrastructureFactory` and asking it for the view. That is what the
`Environment` + `InfrastructureFactory.construct` lines below are — not incidental
plumbing, but the backend selection described in Step 3, now written out explicitly.
Naming the `Environment` after `{SliceName}Projection.name` is what activates the
`upper_snake_case({SliceName})_PERSISTENCE_MODULE` scoping.

Subclass it to keep the view reachable the way `ProjectionRunner` would:

```python
class {SliceName}Runner(BaseProjectionRunner[DcbApplication]):
    """..."""

    def __init__(
        self,
        view: {SliceName}View,
        app: DcbApplication,
        projection: {SliceName}Projection,
    ) -> None:
        self.view = view
        self.projection = projection
        super().__init__(
            projection=projection,
            app=app,
            tracking_recorder=view,
            topics=projection.topics,
        )


def create_runner(
    view_class: type[{SliceName}View] = POPO{SliceName}View,
    env: EnvType | None = None,
) -> {SliceName}Runner:
    """..."""
    environment = Environment(
        {SliceName}Projection.name,
        {**os.environ, **(env or {})},
    )
    factory: InfrastructureFactory[{SliceName}View] = (
        InfrastructureFactory.construct(env=environment)
    )
    view = factory.tracking_recorder(view_class)
    app: DcbApplication = DcbApplication(env=env)

    def command(entry: {EntryName}) -> None:
        app.do({CommandSliceName}(...))

    # `failure` and the `failure=` argument below are BOTH omitted when the
    # model defines no failure event; `_no_failure` is the default (Step 4b).
    def failure(entry: {EntryName}, reason: str) -> None:
        app.do(Record{FailureEventName}Slice(..., reason=reason))

    projection = {SliceName}Projection(
        view=view,
        command=command,
        failure=failure,
    )
    # Recover work orphaned by an earlier crash before the subscription resumes;
    # those events are past max_tracking_id and will never be redelivered.
    projection.drain()
    return {SliceName}Runner(view=view, app=app, projection=projection)
```

The closure over `app` is the whole trick: the projection reaches the write side
through a callable it neither owns nor can introspect.

`drain()` must run **before** the runner is constructed — constructing it opens the
subscription and starts the processing thread.

---

## Step 6 — Acceptance tests

File: `tests/acceptance/snake_case({Context})/snake_case({SliceName})/test_snake_case({SliceName}).py`

GWT cannot drive a `Projection` (see `CLAUDE.md`), so construct the view and
projection directly and call `process_event` yourself. Pass a recording fake as
the command port — no app, no runner, no thread:

```python
class _Recorder:
    """Command port that records each call along with its ambient metadata."""

    def __init__(self) -> None:
        self.calls: list[tuple[{EntryName}, dict[str, str]]] = []

    def __call__(self, entry: {EntryName}) -> None:
        self.calls.append((entry, get_metadata_from_context()))
```

Capturing `get_metadata_from_context()` **at call time** is what lets you assert
on propagation; the context is unwound by the time the test body resumes.

Build the pair in a fixture and drive it directly:

```python
@pytest.fixture
def recorder() -> _Recorder:
    """Return a command port that records its calls."""
    return _Recorder()


@pytest.fixture
def projection(recorder: _Recorder) -> {SliceName}Projection:
    """Return a projection over a fresh in-memory view."""
    return {SliceName}Projection(view=POPO{SliceName}View(), command=recorder)


def test_trigger_fires_the_command(
    projection: {SliceName}Projection, recorder: _Recorder,
) -> None:
    """The trigger records an entry and fires the command."""
    envelope = TaggedEvent(
        decision={TriggerEventName}(entity_id="a", field="value"),
        tags=["{{entity_kind}}:a"],
        metadata={"correlation_id": "corr-1"},
    )
    projection.process_event(envelope, Tracking("upstream", 1))

    entry, metadata = recorder.calls[0]
    assert entry.entity_id == "a"
    assert metadata["causation_id"] == str(envelope.uuid)
    assert metadata["correlation_id"] == "corr-1"
    assert projection.view.get_entries() == [entry]


def test_emitted_event_drains_the_entry(projection: {SliceName}Projection) -> None:
    """The event the command emits removes the outstanding entry."""
    projection.process_event(
        TaggedEvent(
            decision={TriggerEventName}(entity_id="a", field="value"),
            tags=["{{entity_kind}}:a"],
        ),
        Tracking("upstream", 1),
    )
    projection.process_event(
        TaggedEvent(
            decision={EmittedEventName}(entity_id="a"),
            tags=["{{entity_kind}}:a"],
        ),
        Tracking("upstream", 2),
    )
    assert projection.view.get_entries() == []
```

Cover, in the same style:
- a trigger with no `correlation_id` still fires cleanly (the `suppress(KeyError)` path)
- **a command that raises does not propagate**, and leaves the entry in place
- `drain()` re-fires an entry seeded straight into the view (the orphan case), skips one already past `MAX_ATTEMPTS`, and is a no-op on a clean view
- when a failure event exists: the failure port is called on failure, the failure event drives another attempt, attempts stop at `MAX_ATTEMPTS`, and the exhausted entry stays parked

### Acceptance-test notes

- **Use strictly increasing `Tracking` notification ids** — pick any
  `context_name` string (consistent within a test) and never reuse an id; reuse
  raises `IntegrityError` from `_assert_tracking_uniqueness`.
- **Always test against `POPO{SliceName}View`,** even when the slice also ships a
  Postgres implementation. It needs no configuration, no server, and no cleanup.
  The behaviour under test is the projection's dispatch logic, which is
  backend-independent by construction.
- **This bypasses `topics` filtering entirely.** Calling `process_event` directly
  means the test is responsible for only sending events the projection is meant
  to handle. A test asserting the projection *ignores* an unrelated event type is
  still valid and useful — send a `TaggedEvent` for a type not in the `match`,
  and assert `view.max_tracking_id(...)` still advanced (proving the `case _:`
  branch tracked it) while the entries stayed unaffected.

---

## Step 7 — Integration tests

File: `tests/integration/snake_case({Context})/test_snake_case({SliceName}).py`

Drive the real `create_runner()` as a context manager, and seed history as **raw
`TaggedEvent`s** through the runner's own application — never by driving another
slice. `ProjectionRunner`-style runners construct their own `DcbApplication`, so
`runner.app` is the only store the subscription can see.

```python
@pytest.fixture
def trigger(runner: {SliceName}Runner, entity_id: str) -> TaggedEvent[Decision]:
    """Seed the trigger event and wait for the automation to settle."""
    seed = TaggedEvent(
        decision={TriggerEventName}(entity_id=entity_id, field="value"),
        tags=[f"{{entity_kind}}:{entity_id}"],
        metadata={"correlation_id": "corr-1"},
    )
    position = runner.app.events.append(events=[seed])
    runner.wait(notification_id=position + 1, timeout=5)
    return seed
```

Three things reliably bite:

**Wait one position past the seed.** `append()` returns the position of the event
it wrote; the automation appends its own event immediately after, so
`position + 1` covers the full feedback loop and the assertions see a settled
view. Never `time.sleep`.

**Tag the seed for the command's boundary.** The seed must carry every tag the
command slice's `consistency_boundary()` selects on. If the selector's tags are
not a subset of the trigger's tags, the replay misses the event and the command
runs against a history in which it never happened.

**Set `metadata` on the seed to test propagation.** `metadata` and `uuid`
round-trip through the store, so a seeded `correlation_id` is what the
projection reads back — and `seed.uuid` is the value the emitted event's
`causation_id` must equal. Returning the whole `TaggedEvent` from the fixture is
what makes that assertion possible:

```python
def test_carries_causation(runner, trigger) -> None:
    """The command names the trigger as its cause."""
    emitted = ...  # read the appended event back off runner.app.events
    assert emitted.metadata["causation_id"] == str(trigger.uuid)
    assert emitted.metadata["correlation_id"] == "corr-1"
```

Seeding is unconditional — `append()` builds a DCB append condition only when
`cb`/`after` is given — so repeated seeds into the same entity's tags all
succeed. That is what removes the need for a test-only seeding `Slice`, its
hand-scoped `consistency_boundary()`, and the `execute()`-before-`save()` step
that a `Slice`-based seed would require.

Also assert here that entities are handled independently — GWT cannot prove
cross-entity isolation.

**Prove the recovery paths against real infrastructure too.** This test applies to
every automation, and it is the one case that must **not** use the `runner`
fixture: build the view and app by hand, consume the position *before* any runner
exists, then start one.

```python
def test_drain_recovers_an_orphaned_entry(entity_id: str) -> None:
    """An entry left outstanding by a crash is re-fired on restart."""
    view, app = POPO{SliceName}View(), DcbApplication()
    position = app.events.append(
        events=[
            TaggedEvent(
                decision={TriggerEventName}(entity_id=entity_id, field="value"),
                tags=[f"{{entity_kind}}:{entity_id}"],
            ),
        ],
    )
    # A crashed run: entry and tracking committed, command never landed.
    view.add_entry(
        {EntryName}(entity_id=entity_id, field="value"),
        Tracking(app.context_name, position),
    )

    projection = {SliceName}Projection(view=view, command=...)
    projection.drain()
    with {SliceName}Runner(view=view, app=app, projection=projection) as runner:
        runner.wait(notification_id=position + 1, timeout=5)
        assert view.get_entries() == []
```

**Seeding into a live runner's view raises `IntegrityError`.** The subscription
processes that same event and calls `add_entry` with the same `Tracking`, so
whichever write loses the race trips `_assert_tracking_uniqueness` — on the
background thread, surfacing later out of `wait()` rather than at the call. The
ordering above is also the more faithful simulation: a crash consumes the
position *before* the process restarts, which is exactly why `drain()` has to run
before the runner is constructed (Step 5).

For the retry loop — **only when the model has a failure event** — run a persistently
failing command under a real runner and wait for `position + MAX_ATTEMPTS`, since each
attempt appends one failure event.

---

## Files to create

```
src/snake_case({ProjectName})/snake_case({Context})/snake_case({SliceName})/
    __init__.py
    projection.py                     # view, projection, runner, create_runner
    slice.py                          # only if the model defines a failure event

tests/acceptance/snake_case({Context})/snake_case({SliceName})/
    test_snake_case({SliceName}).py   # direct process_event calls, fake ports, drain()

tests/integration/snake_case({Context})/
    test_snake_case({SliceName}).py   # real runner, seeded events, wait(), recovery
```

No `routes.py`. No `__init__.py` under `tests/`.

The command slice under
`src/snake_case({ProjectName})/snake_case({Context})/snake_case({CommandSliceName})/` is
**not** listed here — it is build-state-change's artifact (Step 2), with its own tests.

---

## Checklist

- [ ] `Projection.name` set explicitly (it scopes env vars and table names)
- [ ] `topics` includes **both** the trigger event and the event the command emits
- [ ] The command port is injected via `__init__`, with a no-op default
- [ ] `add_entry(..., tracking)` runs **before** the command is fired
- [ ] Every `process_event` branch persists tracking, including `case _:`
- [ ] The command call is wrapped in `try/except Exception` + `logger.exception(...)`
- [ ] `_fire` takes `metadata: dict[str, str]`; every call site derives it (`_causation_metadata(envelope)`, or `{}` from `drain()`)
- [ ] `drain()` implemented, and called in `create_runner()` **before** the runner is constructed
- [ ] Entries carry `attempts`; firing stops past `MAX_ATTEMPTS` and the entry stays parked
- [ ] `get_entries()` takes no argument and returns copies, so callers cannot mutate view state
- [ ] `add_entry` / `remove_entry` write the entry and its `Tracking` in one atomic step
- [ ] `view_class` subclasses the recorder matching the configured `PERSISTENCE_MODULE` (POPO unless durability was asked for)
- [ ] If the model has a failure event: it is in `topics`, injected as a separate optional port, its recording is itself guarded, and its slice's boundary can never match
- [ ] If the model has **no** failure event: no failure port, no third topic, no `_retry` branch — `drain()` alone is the recovery path
- [ ] `correlation_id` carried forward (suppressed if absent); `causation_id` set to `str(envelope.uuid)`
- [ ] Acceptance tests call `process_event` directly with strictly increasing tracking ids
- [ ] Integration tests seed raw `TaggedEvent`s via `runner.app.events.append(...)`, and `wait()` on the returned position rather than sleeping
- [ ] The seeded event carries every tag the command slice's boundary selects on
- [ ] The `drain()` recovery test builds its own view and app, consuming the position **before** any runner is constructed
- [ ] No `routes.py` created (automations are not exposed via HTTP)
