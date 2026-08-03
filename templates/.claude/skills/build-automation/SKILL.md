---
name: build-automation
description: Implements a pyeventsourcing automation slice (tracking view + Projection that fires a command, pytest tests) from a slice.json definition
---

# Build Automation Slice

> Before doing anything else, read the slice definition from `.build-kit/.slices/{Context}/{SliceName}/slice.json` (both segments in their disk form — i.e. `snake_case({Context})/snake_case({SliceName})`). This file is the **source of truth** for which trigger event drives the automation and what command it fires. Never invent fields not defined there.

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
something actually reads that ledger back. See Step 5.

This skill assumes the materialized-view mechanics from **build-state-view**
(`references/materialized.md`): the abstract `{SliceName}View(TrackingRecorder)`
interface, the POPO implementation, atomic entry+tracking writes, and backend
selection. Read that first; this skill covers only what an automation adds.

---

## Step 1 — Read the slice.json

Extract:
- **sliceName** — the automation's name
- **context** — bounded context
- **processors[]** — the automation element. Its `dependencies` name the inbound read models and the **OUTBOUND `COMMAND`** it fires.
- **commands[]** — the command's data fields
- **events[]** — the events that command emits (these close the loop; see Step 3). Check for a **failure event** among them — a "… failed" outbound event enables the retry loop in Step 4b. Its absence is not a blocker; `drain()` covers recovery either way.
- **specifications[]** — test scenarios. Specs expecting the failure event are describing the automation's retry behaviour, not just the command's validation.

Placeholder grammar and the snake_case / kebab-case transforms are identical to
**build-state-view** — see that skill's Step 1 table.

---

## Step 2 — Build the command handler

Follow **build-state-change** to create `src/snake_case({Context})/events.py` and
`src/snake_case({Context})/snake_case({CommandSliceName})/slice.py`. The automation
reuses that `Slice` unchanged; the only difference is what invokes it.

**Do NOT create a `routes.py`** for the automation itself — it is not driven over HTTP.

---

## Step 3 — Create `projection.py`

File: `src/snake_case({Context})/snake_case({SliceName})/projection.py`

### The view

An abstract `{SliceName}View(TrackingRecorder)` plus a `POPO{SliceName}View`, exactly
as in build-state-view — but the automation needs a **`remove_entry`** alongside
`add_entry`, because entries are transient work items rather than accumulated state.

### topics — list the emitted event too

```python
topics = (get_topic({TriggerEventName}), get_topic({EmittedEventName}))
```

The emitted event is not optional. It is what drains the entry once the command
succeeds. Omit it and the view grows forever, holding every licence it ever
cancelled.

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
                self._fire(entry, envelope)
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
    def _fire(self, entry: {EntryName}, envelope: TaggedEvent[Decision]) -> None:
        """Issue the command, carrying causation forward and swallowing failures."""
        metadata = {}
        with contextlib.suppress(KeyError):
            metadata["correlation_id"] = envelope.metadata["correlation_id"]
        metadata["causation_id"] = str(envelope.uuid)
        try:
            with put_metadata_in_context(metadata):
                self._command(entry)
        except Exception:
            # An escaping exception permanently kills the runner's processing
            # thread, stalling every later event. Log and leave the entry behind.
            logger.exception("failed to ... for %s", entry.entity_id)
```

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

Keep `_fire` taking a plain `metadata: dict[str, str]` rather than an envelope.
Recovery (Step 5) fires commands with no triggering event, and there is no
causation to invent for those.

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

### 4b. Failure events — only when the model defines one

If `slice.json` lists a failure event among the command's outbound dependencies
(e.g. `"License cancellation failed"`), record it and add it to `topics`. The
failure then returns through the same subscription and the retry becomes ordinary
event processing — no timer, no polling, no second thread:

```python
            case {FailureEventName}(entity_id=entity_id):
                self._retry(entity_id, tracking, envelope)
```

Inject it as a **separate optional port** with a no-op default, so the projection
works unchanged for models that have no such event:

```python
def _no_failure(entry: {EntryName}, reason: str) -> None:
    """Discard the failure — the default for models with no failure event."""
```

Appending the failure event can itself fail, so wrap that call in its own
`try/except` too — otherwise the recovery path becomes a new way to kill the runner.

**The slice that records a failure needs a boundary that never matches.** Recording
a failure is an observation, not a decision: the same entity may fail repeatedly. A
DCB boundary is an append *condition*, so a selector matching prior failures makes
the second recording raise `IntegrityError`. Select a type the slice never emits:

```python
        return Selector(types=[{SomeOtherEvent}], tags=["never:matches"])
```

Note `Selector(types=[], tags=[...])` is **not** "no boundary" — it means "fail if
any event with these tags exists", which is exactly the collision to avoid.

### 4c. Attempt counting

Carry `attempts: int = 0` on the entry and increment it on every fire. Past
`MAX_ATTEMPTS`, stop firing and leave the entry **parked** — still visible in the
view for inspection, rather than retried forever. Without this, a permanently
failing command loops indefinitely between the failure event and the retry.

Have `get_entries()` return copies (`replace(entry)`), or callers will mutate view
state through the returned dataclasses.

---

## Step 5 — Assemble the runner

The stock `ProjectionRunner` cannot inject the command port, so build the pieces
explicitly around `BaseProjectionRunner`, which accepts an already-constructed
projection. Subclass it to keep the view reachable the way `ProjectionRunner`
would:

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

    def failure(entry: {EntryName}, reason: str) -> None:
        # Omit entirely when the model defines no failure event.
        app.do(Record{FailureName}Slice(..., reason=reason))

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

Cover:
- the trigger records an entry, and fires the command
- the emitted event drains the entry
- `causation_id` equals `str(envelope.uuid)`, and `correlation_id` is carried forward
- a trigger with no `correlation_id` still fires cleanly
- **a command that raises does not propagate**, and leaves the entry in place
- `drain()` re-fires an entry seeded straight into the view (the orphan case), skips one already past `MAX_ATTEMPTS`, and is a no-op on a clean view
- when a failure event exists: the failure port is called on failure, the failure event drives another attempt, attempts stop at `MAX_ATTEMPTS`, and the exhausted entry stays parked

Use strictly increasing `Tracking` notification ids — reuse raises `IntegrityError`.

---

## Step 7 — Integration tests

File: `tests/integration/snake_case({Context})/test_snake_case({SliceName}).py`

Drive the real `create_runner()` as a context manager. Two things reliably bite:

**`execute()` before `save()`.** `repository.save()` only appends decisions the
slice has already collected — on a freshly constructed slice it returns `0` and
nothing is recorded. `app.do()` normally calls `execute()` for you; when seeding
via `save()` to capture the append position, call it yourself:

```python
seed = _Seed{TriggerEventName}(...)
seed.execute()
position = runner.app.repository.save(seed)
runner.wait(notification_id=position + 1, timeout=5)
```

Wait one position **past** the seeded event: the automation appends its own event
immediately after, so `position + 1` covers the full feedback loop and the
assertions see a settled view. Never `time.sleep`.

**Tag the seed for the command's boundary.** The seeding slice must carry every
tag the command slice's `consistency_boundary()` selects on. If the selector's
tags are not a subset of the trigger's tags, the replay misses the event and the
command runs against a history in which it never happened.

Also assert here: correlation/causation survive the real store round-trip, and
entities are handled independently — GWT cannot prove cross-entity isolation.

**Prove the recovery paths against real infrastructure too.** Simulate a crash by
seeding an event, then calling `view.add_entry(entry, Tracking(app.context_name,
position))` directly — that consumes the position exactly as a crashed run would —
and assert `drain()` re-fires it. For the retry loop, run a persistently failing
command under a real runner and wait for `position + MAX_ATTEMPTS`, since each
attempt appends one failure event.

---

## Files to create

```
src/snake_case({Context})/snake_case({SliceName})/
    __init__.py
    projection.py                     # view, projection, runner, create_runner
    slice.py                          # only if the model defines a failure event

tests/acceptance/snake_case({Context})/snake_case({SliceName})/
    test_snake_case({SliceName}).py   # direct process_event calls, fake ports, drain()

tests/integration/snake_case({Context})/
    test_snake_case({SliceName}).py   # real runner, seeded events, wait(), recovery
```

No `routes.py`. No `__init__.py` under `tests/`.

---

## Checklist

- [ ] `Projection.name` set explicitly (it scopes env vars and table names)
- [ ] `topics` includes **both** the trigger event and the event the command emits
- [ ] The command port is injected via `__init__`, with a no-op default
- [ ] `add_entry(..., tracking)` runs **before** the command is fired
- [ ] Every `process_event` branch persists tracking, including `case _:`
- [ ] The command call is wrapped in `try/except Exception` + `logger.exception(...)`
- [ ] `drain()` implemented, and called in `create_runner()` **before** the runner is constructed
- [ ] Entries carry `attempts`; firing stops past `MAX_ATTEMPTS` and the entry stays parked
- [ ] `get_entries()` returns copies, so callers cannot mutate view state
- [ ] If the model has a failure event: it is in `topics`, injected as a separate optional port, its recording is itself guarded, and its slice's boundary can never match
- [ ] `correlation_id` carried forward (suppressed if absent); `causation_id` set to `str(envelope.uuid)`
- [ ] Acceptance tests call `process_event` directly with strictly increasing tracking ids
- [ ] Integration tests call `execute()` before `save()`, and `wait()` rather than sleep
- [ ] The seeding slice carries every tag the command slice's boundary selects on
- [ ] No `routes.py` created (automations are not exposed via HTTP)
