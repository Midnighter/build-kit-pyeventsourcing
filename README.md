# build-kit-pyeventsourcing

Real-time Claude agent + skill kit for the [Eventmodelers](https://eventmodelers.ai) platform. Connect your board to a fully autonomous coding agent that picks up slice status changes, implements the code, and marks work done — all without manual intervention.

---

## How it works

```
Board (Eventmodelers)
  │  slice status → "Planned"
  ▼
Realtime Agent  ──────────────────► tasks.json
  │  listens on Supabase channel         │
  │  writes task on slice:changed        │
  ▼                                      ▼
ralph.sh loop ◄────────────────── Phase 1: load slice
  │  checks tasks.json every 3s          reads task → runs /connect + /load-slice
  │                                      fetches slice definition to .slices/
  │                                      removes task from tasks.json
  │
  ▼
Phase 2: build slice
  checks .slices/**/index.json for status "Planned"
  → sets status "InProgress" on board
  → runs /build-state-change, /build-state-view, or /build-automation
  → runs quality checks (build + test)
  → commits, merges to main
  → sets status "Done" on board
  → waits for the next slice
```

---

## Step 1 — Install

Run the installer in your project directory:

For now

```sh
git clone https://github.com/Midnighter/build-kit-pyeventsourcing
```

Soon, you will be able to use the [eventmodelers CLI](https://github.com/Nebulit-GmbH/Eventmodelers-Build-Kits/blob/main/eventmodelers-cli/README.md):

```sh
npx @eventmodelers/cli init --stack pyeventsourcing
```
