---
name: learn-eventmodelers-api
description: Teaches an agent everything about the eventmodelers platform API — all endpoints, their purpose, request payloads, response shapes, authentication, and element types.
---

# Eventmodelers Platform API Reference

You now have complete knowledge of the eventmodelers platform API. Use this reference whenever you need to call, implement, or reason about any endpoint.

## Architecture Overview

- **Framework**: FastAPI + `pyeventsourcing` (event sourcing)
- **Event sourcing library**: `eventsourcing.dcb.domain.Slice` (Dynamic Consistency Boundary)
- **Event store**: `eventsourcing.dcb.persistence.DCBEventStore`
- **Storage / Auth**: User can choose between pyeventsourcing's supported persistence backends. For the moment, there is no authN/Z
- **Route discovery**: FastAPI routers imported from `src/{context}/{slice_name}/routes.py` and mounted on the top-level app
- **Base URL** (local): `http://localhost:3000`

## Element Types

```python
MODEL_CONTEXT  # Context/domain modeling container
CHAPTER        # Timeline/sequence container
ACTOR          # System participant (swimlane label)
AUTOMATION     # Automated action
API            # External service
SCREEN         # UI screen
COMMAND        # State-changing operation
EVENT          # Domain event
SPEC_ERROR     # Error scenario
TABLE          # Data table
READMODEL      # Query result / materialized view
SCENARIO       # GWT scenario
LANE           # Timeline row
SLICE_BORDER   # Slice boundary marker
```

## Standard HTTP Status Codes

| Code | Meaning |
|---|---|
| 200 | OK with data |
| 201 | Created |
| 204 | No content |
| 400 | Validation error / bad input |
| 401 | Authentication required |
| 404 | Resource not found |
| 409 | Conflict (e.g. duplicate) |
| 500 | Server error |


## 1. Boards

**File**: `src/boards/routes.py`

### POST `/api/org/:orgId/boards/:boardId/events`
Persist board/timeline row events as an array of mixed event types.

**Request body**: Array of node, comment, edge, or board events  
**Response**: `200` — processed results array


### GET `/api/boards`
List all boards.

**Response**: `200` — `Board[]`


### DELETE `/api/org/:orgId/boards/:boardId`
Delete a board.

**Response**: `204`


### GET `/api/org/:orgId/boards/:boardId/events/search`
Search events by node name.

**Query params**: `name` (string)  
**Response**: `200` — matching event array


### GET `/api/org/:orgId/boards/:boardId/events`
Get all board events in sequence.

**Response**: `200` — event array


### GET `/api/org/:orgId/boards/:boardId/nodes/:nodeId/comments`
Get all comments for a node.

**Response**: `200` — comment array


### POST `/api/org/:orgId/boards/:boardId/bucket`
Create a Supabase storage bucket for the board.

**Response**: `200` — `{ ok: boolean, bucket: string, alreadyExisted: boolean }`


## 2. Chapters & Timelines

**File**: `src/chapters/routes.py`

### POST `/api/org/:orgId/boards/:boardId/chapters`
Create a chapter node.

**Request body**: `{ position?: { x: number, y: number } }`  
**Response**: `200` — chapter data


### POST `/api/org/:orgId/boards/:boardId/timelines/:timelineId/columns`
Add a column to a timeline.

**Request body**: `{ index?: number }` (integer index, optional)  
**Response**: `200` — `{ columnId: string, index: number, totalColumns: number }`


### DELETE `/api/org/:orgId/boards/:boardId/timelines/:timelineId/columns/:columnId`
Delete a column from a timeline. Removes the column and all its cells. Cannot delete the last column.

**Response**:
- `200` — `{ columnId: string, totalColumns: number }`
- `400` — validation error (e.g. last column)
- `404` — timeline or column not found


### POST `/api/org/:orgId/boards/:boardId/timelines/:timelineId/lanes`
Add a lane (row) to a timeline.

**Request body**:
```python
from typing import Literal
from pydantic import BaseModel

class LaneRequest(BaseModel):
    type: Literal["actor", "interaction", "swimlane", "spec", "feedback"]
    label: str | None = None
    index: int | None = None
    height: int | None = None
```
**Response**: `200` — lane data


### POST `/api/org/:orgId/boards/:boardId/timelines/:timelineId/cells/:cellId/drop`
Drop a node into a timeline cell. Validates placement rules.

**Request body**: `{ nodeId: string, nodeType: ElementType }`

**Placement rules**:
- `swimlane` lane → accepts `EVENT`
- `interaction` lane → accepts `COMMAND`, `READMODEL`
- `actor` lane → accepts `SCREEN`, `AUTOMATION`
- `feedback` lane → accepts markdown
- `spec` lane → accepts `SPEC_NODE`

**Response**:
- `200` — drop result
- `400` — placement violation
- `404` — cell or node not found


## 3. Nodes

**File**: `src/nodes/routes.py`

All node endpoints require header: `x-user-id`

### POST `/api/org/:orgId/boards/:boardId/nodes/events`
Submit node change events.

**Request body**: `list[NodeChangeEvent]`

```python
from typing import Any, Literal
from pydantic import BaseModel

class NodeEdge(BaseModel):
    id: str
    source: str
    target: str
    sourceHandle: str | None = None
    targetHandle: str | None = None

class NodeData(BaseModel):
    id: str
    data: dict[str, Any]  # backgroundColor, title, type, url, and other node data fields

class NodeMeta(BaseModel):
    type: str  # ElementType
    title: str | None = None
    description: str | None = None
    fields: dict[str, Any] | None = None

class NodeChangeEvent(BaseModel):
    id: str                                # uuid
    eventType: Literal["node:created", "node:changed", "node:deleted"]
    nodeId: str
    boardId: str
    timestamp: int                         # unix ms
    userId: str | None = None
    hash: str | None = None                # content hash
    changedAttributes: list[str] | None = None  # dot-paths e.g. 'meta.title'
    node: NodeData | None = None
    meta: NodeMeta | None = None
    edges: list[NodeEdge] | None = None
    chapterId: str | None = None           # for cell placement
    cellName: str | None = None            # spreadsheet-style e.g. "B2"
```

**Response**: `200` — `{ hashes: { [eventId: string]: string } }`


### GET `/api/org/:orgId/boards/:boardId/nodes`
List all nodes on a board.

**Query params**: `type?: ElementType`  
**Response**: `200` — node record array


### GET `/api/org/:orgId/boards/:boardId/nodes/:nodeId`
Get a single node.

**Response**: `200` — node record OR `404`


## 4. Images

**File**: `src/images/routes.py`

### POST `/api/org/:orgId/boards/:boardId/images/:imageId`
Update a board image.

**Request**: `multipart/form-data` — field `file` (binary)  
**Response**: `204`


### POST `/api/org/:orgId/boards/:boardId/imagesnapshots/:imageId`
Update an image snapshot.

**Request**: `multipart/form-data` — field `file` (binary)  
**Response**: `204`


### POST `/api/org/:orgId/boards/:boardId/image-nodes/:nodeId`
Create an image node.

**Request**: `multipart/form-data` — fields: `file`, `chapterId`, `cellName`  
**Response**: `204`


### POST `/api/org/:orgId/boards/:boardId/images/:imageId/sketch`
Render a sketch description to WebP and upload.

**Request body**:
```python
from typing import Any
from pydantic import BaseModel

class SketchRequest(BaseModel):
    elements: list[dict[str, Any]]         # sketch element descriptors
    semanticDescription: str | None = None # human-readable description stored in metadata
```
**Response**: `204`


### POST `/api/org/:orgId/boards/:boardId/image-nodes/:nodeId/sketch`
Create a SCREEN node from a sketch description.

**Request body**:
```python
from typing import Any
from pydantic import BaseModel

class SketchDescription(BaseModel):
    elements: list[dict[str, Any]]

class SketchNodeRequest(BaseModel):
    chapterId: str
    cellName: str
    description: SketchDescription
    semanticDescription: str | None = None
```
**Response**: `204` OR `400` (validation error)


## 5. Slices

**File**: `src/slices/routes.py`

### POST `/api/org/:orgId/boards/:boardId/timelines/:timelineId/slices`
Create a complete slice (1 column + 3 nodes automatically placed).

**Request body**:
```python
from typing import Any, Literal
from pydantic import BaseModel

class SliceNodes(BaseModel):
    actor: dict[str, Any] | None = None       # partial NodeData
    interaction: dict[str, Any] | None = None # partial NodeData
    swimlane: dict[str, Any] | None = None    # partial NodeData

class CreateSliceRequest(BaseModel):
    type: Literal["state-change", "state-view", "automation"]
    index: int | None = None
    nodes: SliceNodes | None = None
```

**Slice node mapping**:
- `state-change` → SCREEN (actor) + COMMAND (interaction) + EVENT (swimlane)
- `state-view` → SCREEN (actor) + READMODEL (interaction) + EVENT (swimlane)
- `automation` → AUTOMATION (actor) + COMMAND (interaction) + EVENT (swimlane)

**Response**: `200` — slice data


## 6. Specifications (GWT Scenarios)

**File**: `src/specs/routes.py`

### POST `/api/org/:orgId/boards/:boardId/contexts/:contextName/slices/:sliceName/scenarios`
Append a Given-When-Then scenario to a spec node.

**Request body**:
```python
from typing import Any
from pydantic import BaseModel

class ScenarioRequest(BaseModel):
    id: str
    title: str
    vertical: bool | None = None
    examples: list[Any] | None = None
    given: list[str]   # nodeIds — must be EVENTs from same timeline
    when: list[str]    # nodeIds — at most one COMMAND; empty if then has READMODEL
    then: list[str]    # nodeIds — EVENTs only OR exactly one READMODEL (not mixed)
```

**Validation rules**:
- `given`: only EVENTs from same timeline
- `when`: max one COMMAND; must be empty when `then` contains a READMODEL
- `then`: all EVENTs OR exactly one READMODEL — never mixed
- All referenced nodes must belong to the same chapter/timeline

**Response**:
- `201` — `{ scenario, scenarios, specNodeId, isNewNode: boolean }`
- `400` — validation error
- `404` — context or slice not found
- `409` — duplicate scenario title


### GET `/api/org/:orgId/boards/:boardId/contexts/:contextName/spec-info`
Get valid elements for a context (by name lookup).

**Response**: `200` — `{ chapterId: string, elements: ElementRecord[] }`


### GET `/api/org/:orgId/boards/:boardId/contexts/:contextName/slices/:sliceName/spec-info`
Get valid elements for a specific slice.

**Response**: `200` — `{ chapterId: string, elements: ElementRecord[] }`


## 7. Config Import

**File**: `src/config_import/routes.py`

### POST `/api/org/:orgId/boards/:boardId/import-config`
Import an EventModelingJson config to populate a board.

**Request**: `multipart/form-data` with field `file` OR `application/json` body:
```python
from typing import Any
from pydantic import BaseModel

class ImportConfigRequest(BaseModel):
    slices: list[dict[str, Any]]  # SliceDefinition[]
```

**Response**: `200` — transformed canvas with nodes and edges


## 8. Slice Data

**File**: `src/slicedata/routes.py`

### GET ` `
Build structured slice data from board state.

**Query params** (one required): `contextId` OR `contextName`; optional: `sliceId`  
**Response**: `200` — slice data matching event modeling schema


### GET `/api/org/:orgId/boards/:boardId/slicedata/slices`
List all slices on a board.

**Response**: `200` — `{ slices: Array<{ id: string, title: string, status: string }> }`


## 9. Extensions

**File**: `src/extensions/routes.py`

### GET `/api/org/:orgId/boards/:boardId/extensions`
List extension configs for a board.

**Response**: `200` — extension record array


### PUT `/api/org/:orgId/boards/:boardId/extensions/:type`
Enable or disable an extension.

**Request body**: `{ enabled: boolean, config?: object }`  
**Response**: `200` — updated extension config


## 10. Snapshots

**File**: `src/snapshots/routes.py`

All snapshot endpoints require Supabase JWT authentication.

**Constraints**: max 3 snapshots per user, max 30-day retention, max 50 MB file size.

### GET `/api/snapshots`
List current user's snapshots.

**Response**: `200` — `Array<{ id, name, payload_id, expiry, shared }>`


### POST `/api/snapshots`
Create a snapshot.

**Request**: `multipart/form-data` — fields: `payloadFile` (binary), `name` (string), `retention?` (days, max 30)  
**Response**: `201` — `{ ok: true, id: string }`


### GET `/api/snapshots/:id`
Load a snapshot's payload.

**Response**: `200` — snapshot payload JSON


### PATCH `/api/snapshots/:id/share`
Share a snapshot (makes it publicly accessible).

**Response**: `200` — `{ ok: true }`


### DELETE `/api/snapshots/:id`
Delete a snapshot.

**Response**: `200` — `{ ok: true }`


## 11. User Management — Commands (Event Sourced)

All commands respond with:
```python
from typing import Literal
from pydantic import BaseModel

class CommandResponse(BaseModel):
    ok: Literal[True]
    next_expected_stream_version: int
    last_event_global_position: int
```

Optional headers on all: `correlation_id`, `causation_id`

### POST `/api/creategroup`
**Body**: `{ groupId: string, name: string }`  
**Event emitted**: `GroupCreated`


### POST `/api/inviteuser`
**Body**: `{ groupId: string, email: string, invitationId: string }`  
**Event emitted**: `UserInvited`


### POST `/api/acceptinvite`
**Body**: `{ userId: string, groupId: string, invitationId: string }`  
**Event emitted**: `InvitationAccepted`

### POST `/api/assignrole`
**Body**: `{ userId: string, groupId: string, role: string }`  
**Event emitted**: `RoleAssigned`

## 12. User Management — Read Models (Projections)

All require authentication. Optional query param `_id` to filter by ID.

### GET `/api/query/group-details-lookup`
Group details. Filter: `?_id=groupId`

### GET `/api/query/open-invites`
Pending invitations. Filter: `?_id=invitationId`

### GET `/api/query/user-group-assignments`
User-to-group mappings. Filter: `?_id=groupId`

### GET `/api/query/users-to-assign-to-groups`
Users available for group assignment. Filter: `?_id=userId`

## 13. Utility

### GET `/api/user`
Get current authenticated user info.

**Response**: `{ user_id: string, email: string, metadata: object }`

### GET `/api-docs`
Swagger UI (interactive API explorer)

### GET `/swagger.json`
OpenAPI specification (JSON)

## Domain Events

### Snapshot Events (`src/snapshots/events.py`)

```python
SnapshotStored           # { name, id, payloadId, expiry }
SnapshotDeleted          # { id }
SnapshotCleanedUp        # { id }
PublishedSnapshotDeleted # { id }
SnapshotShared           # { id }
SnapshotPublished        # { id, payloadId, bucket, path }
```

### User Management Events (`src/usermanagement/events.py`)

```python
GroupCreated          # { groupId, owner, name }
UserAssignedToGroup   # { groupId, userId }
UserInvited           # { groupId, invitationId, email }
InvitationAccepted    # { invitationId, groupId, userId }
RoleAssigned          # { groupId, userId, role }
```

All events (dataclasses subclassing `eventsourcing.dcb.dataclasses.Decision`) support optional metadata: `user_id`, `correlation_id`, `causation_id`

## Key Source Files

| File | Purpose |
|---|---|
| `src/change/types.py` | `ElementType`, `NodeChangeEvent`, `EdgeEvent` |
| `src/boards/routes.py` | Board CRUD + event persistence |
| `src/chapters/routes.py` | Chapters, columns, lanes, cell drops |
| `src/nodes/routes.py` | Node event sourcing |
| `src/images/routes.py` | Image upload + sketch rendering |
| `src/slices/routes.py` | Slice creation |
| `src/specs/routes.py` | GWT scenario management |
| `src/config_import/routes.py` | Config import |
| `src/slicedata/routes.py` | Slice data read models |
| `src/extensions/routes.py` | Extension management |
| `src/snapshots/routes.py` | Snapshot CRUD |
| `src/usermanagement/{slice_name}/routes.py` | User management commands + projections |
| `src/snapshots/events.py` | Snapshot domain events |
| `src/usermanagement/events.py` | User management domain events |
| `src/main.py` | FastAPI app wiring, CORS, `/api/user` |
