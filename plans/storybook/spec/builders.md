# Builders

## Where they live

The builders live in `fleet`'s spec-support unit, behind the `fleet/testing` entry that already keeps test helpers out
of every production bundle (`web/projects/fleet/src/lib/testing/public-api.ts`). Each lives in
`fleet/src/lib/testing/builders/`, one file per family of wire shapes, and `public-api.ts` re-exports each builder by
name:

| File            | Builds                                                                                                                                                                         |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `chunk.ts`      | `ChunkDetail` and what it carries: `TransitionView`, `WorkRefView`, `ChunkEscalationView`, `ChunkNeighborView`, `RouteView`, `PrView`, `LandedRepoView`, `ChunkUsageTotalView` |
| `artifact.ts`   | `ArtifactView`, one named builder per `ArtifactKind` that carries a structured body, and the finding shapes inside a survey and a delta                                        |
| `transcript.ts` | `TranscriptSegmentIndexEntry`, `TranscriptSegmentContentView`, and the hub's turn, tool-call, and sidechain segment views                                                      |
| `board.ts`      | `ChunkCountsView`, `FleetSpendView`                                                                                                                                            |
| `project.ts`    | `ProjectView`                                                                                                                                                                  |
| `clock.ts`      | `FIXTURE_NOW`, the instant every builder dates its records from                                                                                                                |

`bzh:frontend-package-layers` lets the `testing` unit import `api` and nothing else, and the builders need nothing else:
each imports its types from the generated client, `../../api/hub` and `../../api/runner`. A part typed by one of
`fleet`'s own aliases over the wire takes a builder's output unchanged. `TranscriptViewer`'s `TranscriptTurn`, for one,
is the union of the hub's and the runner's generated turn views. A family grows a file when a story or spec first needs
it; the table above is the set the roster in [stories.md](./stories.md) needs.

## What a builder is

A builder is a function named for the shape it builds, in lower camel case — `chunkDetail`, `artifactView`,
`transitionView` — taking a `Partial` of that shape and returning the whole of it:

```ts
export function chunkDetail(overrides: Partial<ChunkDetail> = {}): ChunkDetail {
  return {
    chunk_id: 'ch_01fixture0000000000000000000',
    graph_id: 'gr_build',
    status: 'running',
    current_node_id: 'nd_build',
    latest_epoch: 1,
    work_refs: [workRefView()],
    history: [transitionView()],
    artifacts: [],
    ...overrides,
  };
}
```

Its return type is the generated type itself, never a copy, so a field the wire adds, renames, or makes required breaks
the builder at compile time, and every story and spec built on it breaks in the same change. A builder fills every
required field and the optional fields a part on the roster reads, with values a person would recognize from a real
fleet: a work item with a plausible title, a graph whose nodes are named as the packaged graphs name theirs, costs and
token counts of a realistic size. Its defaults describe one ordinary record, not an interesting one; a state is the
caller's to compose.

A builder is deterministic. Its ids are fixed strings, its instants are offsets from `FIXTURE_NOW`, and it holds no
counter and reads no clock, so a story renders the same on every visit and a spec's literal never depends on what ran
before it. A caller that needs several distinct records passes their ids. `FIXTURE_NOW` is a fixed UTC instant, and the
workshop provides it as `FLEET_CLOCK` ([toolchain.md](./toolchain.md) §What every story renders inside).

Nested shapes compose by calling a builder inside an override, rather than through deep-merge machinery:

```ts
const escalated = chunkDetail({
  status: 'needs_human',
  escalation: chunkEscalationView({ wrapped_takeover_command: '' }),
});
```

A family may export a few named states the roster asks for repeatedly, such as `landedChunkDetail` or
`bouncedChunkDetail`. Each is a builder over the plain one, with its name saying what is different about it.

## Stories and specs

Stories and specs share the builders and keep their states to themselves. A story file declares the states it shows,
built from the builders, and no spec imports a story's state. A spec builds its own subject from the same builders and
spells out, in its own overrides, every field its assertion depends on, even where the builder's default already holds
the value:

```ts
const detail = chunkDetail({
  escalation: chunkEscalationView({
    takeover_command: 'cd /work/ch_01 && claude --resume se_01',
    wrapped_takeover_command: '',
  }),
});
```

A builder's defaults can then be tuned to make a story look right without moving any spec's ground. A spec that leaned
on a default would be silently re-pointed by that tuning, which is the one coupling the separation exists to prevent.

Specs move onto the builders when a change next touches them, never in a sweep of their own. The specs of the pieces on
the roster are the likeliest first movers, since the stories land beside them.

Whether a spec may take a story as its fixture, so that a state is described once rather than twice, is left open
([the epic plan](../index.md)). Nothing in this epic builds toward it or rules it out.
