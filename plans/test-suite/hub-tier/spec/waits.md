# Waits that end when the system has spoken

The ported tests wait on no clock. `sse_tap`'s `settle` and `SseTap.collect`'s `window` have no counterpart in the
harness, and `waitForTimeout` is already banned in specs
([the lint](../../foundation/spec/rulebook.md#what-the-harness-enforces)).

## Opening the stream

A test opens its stream before it acts, so nothing it waits for can precede the subscription, and there is no replay to
drain:

- **At `fresh-tenant` and `fresh-hub`**, `events.open()` connects with no cursor the moment the tenant exists. The
  tenant is new, so the replay is the tenant's whole history from its first event, and the test reads all of it.
- **At `common` and `fresh-project`**, the tenant's history is every other test's, so `events.open()` connects with
  `from=latest` ([surfaces.md](./surfaces.md#joining-the-stream-from-now)) and reads only frames naming the test's own
  ids.

## Positive assertions

`events.waitFor(predicate)` resolves with the first frame that satisfies it, and `waitForAll(predicates)` once every one
has been met, each with a bound after which the test fails, never a duration it waits out. A test asserting that three
frames arrive returns when the third does.

## The barrier

A negative assertion, such as "exactly one `queue-changed`" or "no frame for the other project", has no event of its own
to wait on. It waits instead on a barrier: a known, harmless act on something only the test made, performed after the
act under test, whose frame the test waits for. The stream is ordered within a tenant, so every frame the act under test
produced arrives before the barrier's, and once the barrier's frame has arrived the test may assert on what came before
it.

`events.open({ barrier: true })` prepares one before subscribing: it files and ingests a barrier chunk through `issue`,
then opens the stream, so the barrier chunk's own creation precedes the subscription and never appears in what the test
reads. `events.barrier()` edits the barrier chunk's title through `PATCH /api/chunks/{chunk_id}`, waits for its
`chunk-changed` frame with cause `edited`, and returns the frames that arrived before it. The reader drops every frame
naming the barrier chunk, so the test sees only its own act's frames. The epic verifies, with a test of the harness's
own, that a title edit emits no frame but that one, `queue-changed` in particular; if it emits more, the reader drops
those too, by their place between the edit's request and its `chunk-changed` frame.

The barrier holds only where the frames under test are published before the request that caused them returns. The hub's
routes publish inside the handler, after the write and before the response (`ChunkChanged.publish` in
`src/blizzard/hub/api/chunks.py`, and likewise in `fleet.py`, `queue.py`, `routines.py`, `work_sources.py`), and the
broker hands an event to every subscriber's queue in id order. Frames a background sweep publishes are not ordered
against a request, so the barrier proves nothing about them. Each negative assertion is checked against the path that
emits its frames as it is ported, and one whose frames come from a sweep waits on the sweep's own completion instead, on
a `fresh-hub` where that line is the test's to read, or keeps a bounded window with a comment saying why.

## The forge-status model case

`test_a_non_opted_in_source_starts_no_sweep_loop` sleeps three seconds and asserts its issue has no forge label. Its
name and docstring say no sweep loop starts, but `HubServices.annotation` is always built
(`src/blizzard/hub/composition.py`) and the loop does start; what the test proves is that a source that has not opted in
is never labelled. On a shared hub it could show only that its own issue went unlabelled, so it takes `fresh-hub`, in
the `annotation` profile's folder.

Its port waits for two `"sweep pass completed"` lines from the `blizzard.hub.forge_status` logger after its issue is
ingested, read from its fresh hub's whole log, so at least one full pass has run over the issue, and then asserts the
issue has no label. The title keeps the original function name, so its citations move mechanically, and the port's
comment says what it actually proves. Renaming it is a separate change, after the move.
