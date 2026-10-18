# Builders

## Where they live

The builders live in `fleet`'s spec-support unit, behind the `fleet/testing` entry that already keeps test helpers out
of every production bundle (`web/projects/fleet/src/lib/testing/public-api.ts`). Each lives in
`fleet/src/lib/testing/builders/`, one file per family of wire shapes, and `public-api.ts` re-exports each builder by
name:

| File            | Builds                                                                                                                                                                         |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `chunk.ts`      | `ChunkDetail` and what it carries: `TransitionView`, `WorkRefView`, `ChunkEscalationView`, `ChunkNeighborView`, `RouteView`, `PrView`, `LandedRepoView`, `ChunkUsageTotalView` |
| `artifact.ts`   | `ArtifactView` of each kind, the `FindingSurvey` and `FindingDelta` an asset's content carries, and the assets that carry them, a bounce envelope among them                   |
| `transcript.ts` | `TranscriptSegmentIndexEntry`, `TranscriptSegmentContentView`, and the hub's turn, tool-call, and sidechain segment views                                                      |
| `board.ts`      | `ChunkCountsView`, `FleetSpendView`                                                                                                                                            |
| `clock.ts`      | `FIXTURE_NOW`, the instant every builder dates its records from                                                                                                                |

`bzh:frontend-package-layers` lets the `testing` unit import `api` and nothing else, and the builders need nothing else:
each imports its types from the generated client, `../../api/hub` and `../../api/runner`. A part typed by one of
`fleet`'s own aliases over the wire takes a builder's output unchanged. `TranscriptViewer`'s `TranscriptTurn`, for one,
is the union of the hub's and the runner's generated turn views. A family grows a file when a story or spec first needs
it; the table above is the set the roster in [stories.md](./stories.md) needs.

## The artifact family

An `ArtifactView` is one of two kinds on the wire, `git_commit` or `asset`. A survey, a delta, and a bounce envelope are
not kinds: each is an asset whose `content` is a JSON string, which `ChunkArtifactBody` parses to choose a view. Survey
and delta are told apart by their shape, and a bounce envelope by the asset's key, `bounce-envelope`. `artifact.ts`
follows the wire rather than the views:

- **One builder per kind.** `assetArtifactView` builds an asset whose content is plain text, and `commitArtifactView` a
  `git_commit` with its repository, commit hash, and branch.
- **The content shapes, typed against the generated client.** `findingSurvey` and `findingCandidate` build a
  `FindingSurvey`; `findingDelta`, with `addFindingOp`, `observedFindingOp`, and `goneFindingOp` for its findings,
  builds a `FindingDelta`. `ChunkArtifactSurvey` and `ChunkArtifactDelta` take these as their inputs, so their stories
  use them as they are.
- **The assets that carry them.** `surveyArtifactView(survey)` and `deltaArtifactView(delta)` build an asset whose
  content is the shape serialized, and `bounceArtifactView(envelope)` an asset keyed `bounce-envelope` whose content
  serializes a plain record of `cause`, `detail`, and any further keys. `ChunkArtifactBody`'s stories build from these,
  so each story goes through the body's real parsing, its fallback to verbatim text included.

No builder makes a `BounceEnvelope`. It is the bounce view's own reading of the content, declared in `chunk/`, which
`testing` may not import, and it has three fields. `ChunkArtifactBounce`'s stories, beside it in `chunk/chunk-detail/`,
write their envelopes as literals.

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
built from the builders. A spec builds its own subject from the same builders and spells out, in its own overrides,
every field its assertion depends on, even where the builder's default already holds the value:

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

Once the builders and the stories both exist, a spec may take a story as its fixture, so that a state is described once
rather than twice ([the epic plan](../index.md)). Whether specs should is the owner's question after living with them,
so this epic builds nothing toward it, and no spec takes a story as its fixture in it.
