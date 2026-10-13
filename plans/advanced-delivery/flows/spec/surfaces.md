# Surfaces

Every surface here reads; nothing an operator does creates, edits, or reorders a flow. Reads take
`Permission.FLEET_VIEW`, as the board does.

## API

A `hub/api/flows.py` router declaring the board's dependencies (`reject_runner_principal`, `tenant_bound`) on itself, as
`hub/api/chunks.py` does, included in `create_app`, with wire models in `wire/flow.py`:

```text
GET /api/flows                       ?active=true
GET /api/flows/{flow}
GET /api/flows/{flow}/record         ?before=&limit=
```

`{flow}` is a `flw_<ulid>`. `GET /api/flows` lists the tenant's flows as `FlowView`: id, resource, repository id, slug
and name, branch, the turn open now (chunk, started at), the line (`FlowPlaceView` per open submission in arrival order:
chunk id, arrival, `state` of `landing`, `next`, `waiting`, or `polling` with its next poll time), and the last verdict
and the last landing. `active=true` keeps flows with an open submission or a fact in the board window's 48 hours, the
board's own default. `GET /api/flows/{flow}` answers one `FlowView`. `/record` pages the flow's facts newest first as
`FlowFactView`, joined to each chunk's short id and title.

The chunk reads gain the chunk's places. `ChunkSummary` and `ChunkDetail` (`wire/chunk.py`) carry
`flow_places: list[ChunkFlowPlaceView]`, one per flow its open submission holds: flow id, resource, repository slug,
branch, `state`, and `position` — one for the chunk landing or next, counting only eligible submissions ahead of it.
`ChunkDetail` also carries `flow_record: list[FlowFactView]`, the facts of every submission the chunk made, which the
detail's timeline interleaves with its transitions and bounces in `deriveHistoryRows`.

The OpenAPI document is re-exported (`blizzard-export-openapi`) and the web client regenerated
(`npm run generate:client`).

## Live channel

`HubEventType` (`foundation/hub_event_types.py`) gains `flow-changed`, published on every submission, turn start, turn
end, landing, and withdrawal with payload `FlowChangedPayload(flow_id, resource, chunk_id, kind)` (`wire/sse.py`), added
to `SSE_FRAME_MODELS`, `HUB_SCHEMA_COMPONENTS`, and the `contracts/sse/` corpus and its `manifest.json`, with a
`publish_flow_changed` in `hub/events/broker.py` and cases in `fleet-live.spec.ts` and `sse-contract.spec.ts`, the path
the `host-changed` frame took. The chunk's own `chunk-changed` frame still fires when its status changes.
`EVENT_INVALIDATION_REGISTRY` (`web/projects/fleet/src/lib/sse/fleet-live.ts`) gains a row that invalidates the flows
query and the named chunk's queries.

## CLI

`hub/cli/flow.py` adds `blizzard hub flow list [--all]` and `blizzard hub flow show <flow> [--record] [--limit N]`.
`blizzard hub chunk show` prints a chunk's places under its status. `contracts/cli/hub.json` is regenerated.

## The board

A board's columns stay its status lanes; `delivering` still sits in RUNNING (`STATUS_LANE`,
`web/projects/fleet/src/lib/core/chunk-lanes.ts`). The flows sit in a **Delivery** band above the columns, one lane per
active flow, in `web/projects/hub/src/app/board/flows/`:

- The lane is titled by its repository's name and branch (`blizzard · master`), and shows, left to right, the chunk
  landing now, the chunks in line in arrival order, and the last verdict with how long ago it came.
- A chunk in line shows its state: next, waiting its poll (with the time of it), or passed over.
- Clicking a chunk opens it in the chunk dock as a card does; clicking the lane title opens the flow's record in the
  dock.
- A lane with nothing landing and no one in line collapses to a single line showing its last verdict, and drops out
  after the board window.
- The band is hidden when no flow is active.

A `delivering` card in RUNNING shows its places in one line under its node — "landing on `blizzard/master`" or "3rd on
`blizzard/master`", one per flow — from `ChunkSummary.flow_places`. `BoardCard` gains `flowPlaces`, and its delivery
block shows whenever the chunk has a place, not only when it has an open pull request or awaits an external merge. The
chunk detail's timeline shows its submission, each turn and how it ended, and each landing, and links each to its flow's
record.
