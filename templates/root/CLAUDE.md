# Project notes for Claude

Conventions and gotchas that apply to every file in this repo. Skills should describe **what** to build; this file records **how** to keep it committable.

## Tooling

- **hatch** manages every Python environment. Never call `python`/`pip`/`pytest` directly — use `hatch -e dev run <cmd>` or `hatch -e dev shell`. Each test suite has its own env: `unit-tests`, `acceptance-tests`, `integration-tests`, `documentation-tests`.
- **pre-commit** runs on every commit. All hooks must pass; the list is authoritative in `.pre-commit-config.yaml`.
- **Commit each unit of work.** When a coherent chunk is done (a slice, a bug fix, a refactor), commit it. `git commit` triggers the pre-commit hooks, which is the intended way to catch header/docstring/style violations — don't rely on eyeballing the compliance rules below. If a hook rewrites files, stage the result and re-run until the commit succeeds.

## Pre-commit compliance rules

Every new Python module must satisfy these before it can be committed:

- **Copyright header** on line 1: `# Copyright {YYYY} Moritz E. Beber` (ruff `CPY001`).
- **Module docstring** immediately after the header (`D100`).
- **Class docstring** on every public class (`D101`).
- **Method/function docstring** on every public callable (`D102`, `D103`, `D104` for packages).
- **`raise ... from exc`** when re-raising inside `except` (`B904`).
- **String exception messages** go into a variable first, then `raise` (`EM101`).
- **Trailing commas** in multi-line calls / literals (`COM812`).
- **Imports sorted** — first-party is `src` / the project package name (`I001`).

## FastAPI / Pydantic gotchas

- **Depend on `Annotated[T, Depends(...)]`, not `T = Depends(...)`.** The old form trips `FAST002` and `B008`.
- **Pydantic model fields need runtime type imports.** `from __future__ import annotations` + PEP 563 is fine for everything *except* types that appear as `BaseModel` fields — Pydantic can't rebuild the schema when the class is under `TYPE_CHECKING`. Import such types at runtime and silence `TC003` with a `# noqa: TC003` comment on that specific line.
- **`lru_cache`'d dependency factories persist across tests.** Integration tests must override them via `app.dependency_overrides[get_x] = lambda: fresh_x`; otherwise state leaks between test cases.

## Test layout

- **No `__init__.py` under `tests/`.** Pytest doesn't need it; adding them is unwanted clutter.
- **`@pytest.fixture`** — no parentheses (`PT001`).
- **`tests/acceptance/`** — slice-level tests using `eventsourcing.dcb.gwt` (given/when/then over an in-memory `DcbApplication`).
- **`tests/integration/`** — API-level tests using `fastapi.testclient.TestClient` against a fresh `FastAPI()` per test.
- **`tests/unit/`** and **`tests/documentation/`** — reserved for their respective purposes; a placeholder test must exist in each so pytest doesn't exit with code 5.
- **`with TestClient(app) as client:`, not a bare `TestClient(app)`,** whenever the app defines a `lifespan`. The `with` block is what drives the lifespan context manager; without it nothing set on `app.state` during startup exists and every route raises `AttributeError`.
- **Never `time.sleep` to wait for a background thread.** `TrackingRecorder.wait(context_name, notification_id, timeout)` and `ProjectionRunner.wait(notification_id, timeout)` exist for this. They block the calling thread, not the event loop — safe to call from a sync test against an async app.
- **Integration tests need `httpx2`** in the `integration` dependency group and a matching `integration-tests` hatch env.

## pyeventsourcing DCB API quick reference

- The class is `DcbApplication` (lowercase `cb`), not `DCBApplication`.
- `Selector(types=[E], tags=[])` — `types` is a `Sequence`, not a set.
- `app.do(slice_instance)` — pass an **instance**, not the class; `do()` internally calls `slice.execute()` with no arguments, so command arguments must live on `self`.
- **`@event` handlers receive only the fields they declare.** The library inspects the handler's signature and passes only the matching kwargs — so `def _(self): ...` and `def _(self, order_id: str): ...` are both valid on the same event type. Declare only the fields the handler actually needs.
- **Selector tags ⊆ trigger tags.** Every tag a `consistency_boundary()` selector asks for must be present on the tags the emitting slice passes to `trigger_event(..., tags=...)`. If the selector's tags aren't a subset, the replay misses the event and the next command (or view) sees a history in which it never happened.
- **`Selector(types=[], tags=[])` is not "no boundary" — it is "fail if *any* event exists anywhere."** An empty selector is a DCB append condition over the whole store, so two writes for unrelated entities in the same test collide with `IntegrityError`. Always scope `types`/`tags` to the entity, including in test-only emitting slices.
- **`repository.save(slice_)` returns the `int` append position** — the same value `TrackingRecorder.wait` polls via `max_tracking_id`. Don't try to read it off `slice_.new_decisions`; `collect_events()` drains that list during `save()`.

### Projections (materialized views)

- **`Projection.topics` is a tuple of topic strings** (`get_topic(EventClass)`), not `Selector` instances. It filters what the subscription pulls before `process_event` is called.
- **`match` on `envelope.decision`, not `envelope`.** `envelope` is a `TaggedEvent[Decision]`; the payload is `envelope.decision`. Keep a `case _:` wildcard even though `topics` already filters — `match` is exhaustive-by-branch, not exhaustive-by-topics.
- **Every `process_event` branch must persist the tracking position** via `add_entry(..., tracking)` or `view.insert_tracking(tracking)`, whether or not the view changed. `wait()` polls `max_tracking_id`, which only advances when something records that `Tracking`; a branch that forgets it makes every later `wait()` hang until timeout.
- **Tracking uniqueness is enforced by the recorder**, not the projection — reusing a `Tracking` notification id raises `eventsourcing.persistence.IntegrityError` from `_assert_tracking_uniqueness`. Use strictly increasing ids across events in a test.
- **Set `Projection.name` explicitly.** It picks prefixed env vars and (for database-backed recorders) table names; the `__init_subclass__` default is the class's own `__name__`.

### Testing DCB code

- GWT test helpers: `given(*TaggedEvent).when(slice_instance).then(*TaggedEvent)`. `then` compares `TaggedEvent` instances, not classes.
- **`given`/`when` only drive `Slice` objects** — they dispatch through `@event`-decorated handlers on the object passed to `when()`. A `Projection` is driven by `process_event` instead, so test it by constructing the view and projection directly and calling `process_event(envelope, Tracking(context_name, notification_id))` yourself. No runner, no background thread.
- **GWT refuses histories outside the consistency boundary.** Prior events on `given()` must carry tags overlapping the slice's `consistency_boundary()`, or `when()` raises `AssertionError("Consistency boundary wouldn't have selected: ...")`. This is deliberate — but it means cross-entity isolation ("another entity's events don't leak into this one") can't be proven at the acceptance level. That property belongs in the integration suite.
