# Notes

Backlog for the build kit itself. Not shipped into generated projects.

## Deferred: adopt `application/problem+json` for domain rejections

From *Rethinking REST API Design* (Rico Fritzsche, 2026-07-09). The URL half of
that article is adopted — see `templates/root/CLAUDE.md` → *API addressing*. The
error half is not, and this is what it would take.

### Where the kit is today

`templates/.claude/skills/build-state-change/SKILL.md` raises untyped
`ValueError`s out of `execute()` and flattens all of them to one status in the
route:

```python
raise ValueError(msg)          # in the slice's execute()

except ValueError as exc:      # in routes.py
    raise HTTPException(
        status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
        detail=str(exc),
    ) from exc
```

The *Error mapping* table in that same skill already promises `409 Conflict` for
"domain conflict class (e.g. duplicate)", but no template produces one — there is
no domain-error class to catch, so every rejection lands on 422 regardless. That
gap is the real motivation here; problem+json is the shape of the fix, not the
whole of it.

### What the article argues

A business rule refusing a command is not a malformed request. `422` says "your
syntax is wrong"; the client sent a perfectly well-formed cancellation request
for a licence that is already cancelled. That is `409 Conflict`, and the body
should be machine-readable (RFC 9457) rather than a prose string a client has to
regex:

```json
{
  "type": "https://example.com/problems/licence-already-cancelled",
  "title": "Licence already cancelled",
  "status": 409,
  "detail": "Licence lic_123 was cancelled on 2026-03-04.",
  "instance": "/licences/lic_123/cancellation-requests"
}
```

The `type` URI is the contract — stable, versioned, documented — and it is what a
client branches on. `title` and `detail` are for humans.

### What adopting it implies for the kit

This is bigger than a response-body change; it is a typed-domain-error refactor:

1. **A domain error base class**, in the shared runtime modules alongside
   `command.py` (created once, never per-slice — same rule as `CommandOutcome`).
   Each slice's rejection reasons become subclasses carrying a stable `type` slug,
   replacing `raise ValueError("already_processed")`.
2. **Acceptance tests change shape.** The GWT tests currently assert
   `pytest.raises(ValueError, match="already_processed")` — matching on a *string*.
   Typed errors make that `pytest.raises(LicenceAlreadyCancelled)`, which is the
   point, but it touches the test template in every slice skill.
3. **One exception handler on the app**, not a `try/except` in every route.
   `create_app()` registers a handler mapping the base class to a
   `JSONResponse(status_code=…, media_type="application/problem+json")`. The
   per-slice `routes.py` template gets *simpler* — the `except ValueError` block
   disappears — which is a good sign for the design.
4. **`type` URIs need a home.** They are public contract, so they belong in the
   generated OpenAPI spec (`responses={409: …}` per route) rather than only in
   prose. Decide whether the slice author writes them or they are derived from
   the error class name.
5. **The 422 does not disappear** — it stays for genuine Pydantic request-shape
   failures, which FastAPI raises before any slice code runs. The distinction
   between the two becomes visible instead of collapsed, which is the actual win.

Open question: whether `slice.json` should carry the rejection reasons (it is the
source of truth for fields and events, so a slice author would reasonably expect
to declare failure modes there too), or whether they stay implicit in `execute()`.

## Deferred: `availableIntentions` on view responses

Also from the same article. A view response advertises which commands are
currently legal for the entity it describes, so the client renders affordances
from the server's answer instead of reimplementing the domain rules:

```json
{
  "dogId": "dog_123",
  "status": "boarding",
  "availableIntentions": [
    {"intention": "check-out", "href": "/dogs/dog_123/check-outs"},
    {"intention": "extend-stay", "href": "/dogs/dog_123/stay-extensions"}
  ]
}
```

This pairs naturally with the URL scheme now adopted: the `href` values are
exactly the intention addresses the state-change skill derives, so the two halves
agree by construction.

### Why it was held back

It introduces a **view → command coupling** the kit currently does not have, and
that coupling is the thing to think through before implementing:

- A view slice today knows only its own events and its own projected state. To
  list available intentions it must know which command slices exist in the
  context and what each one's preconditions are — knowledge that lives in the
  *other* slices' `execute()` methods.
- Duplicating those preconditions in the projection means one rule in two places,
  drifting silently. The whole appeal of the feature is that the client stops
  duplicating the rules; trading a client-side duplicate for a server-side one is
  not obviously a win.
- The alternative is a shared per-context registry of intentions and their
  guards, which both the command slices and the views consult. That is a real
  architectural addition to the kit, not a template tweak, and it cuts across the
  "each slice is self-contained" property the skills currently maintain.
- Because the `href`s are now derivable from boundary tags, a registry could
  plausibly *generate* them rather than have each view hand-write them — worth
  exploring, since it would also let the OpenAPI spec cross-check that every
  advertised `href` is a route that actually exists.

Revisit once a generated project has enough slices in one context to show whether
the duplication is actually painful.
