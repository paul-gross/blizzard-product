# Surfaces

Every surface the flows slice built reads a batch too; nothing an operator does forms, splits, or reorders one. The
batching settings and `forge-queue` are repository fields and travel its doors as the policies slice's fields do.

## API

`FlowView` (`wire/flow.py`) gains `batch: BatchView | None`, the batch in progress: its id, its state, the train branch
and its web address, its base revision and revision, how long it has been in its state, the batch it was split from, and
its members, each a chunk id with its position and whether it is merged onto the train yet. It also gains `last_batch`,
the newest finished batch with how it ended. `FlowPlaceView.state` and `ChunkFlowPlaceView.state` gain `ready`, prepared
and waiting for a batch; `in-batch`, carried by the batch in progress; and `in-forge-queue`, with the forge's position,
for a `forge-queue` repository. `FlowFactView` carries `batch_id`, so `/record` reads a batch's whole history in order.

The repository's wire models gain `batch_size`, `batch_wait_seconds`, and `train_prefix`, and its document the last
probe — the probed capabilities it showed and when it was taken. `landing_policy` admits `forge-queue`, and a write that
falls back answers with a `warnings` entry naming the fallback. The schema lists, for each policy, its fallback. The
OpenAPI document is re-exported and the web client regenerated.

## Live channel

`flow-changed` (`FlowChangedPayload`) is published for every batch fact as for a submission's, carrying `batch_id`,
which `wire/sse.py`, the `contracts/sse/` corpus, and `fleet-live.ts`'s invalidation row learn.

## CLI

`blizzard hub flow show` prints the batch in progress under the line, and `--record` the batch facts with the rest.
`blizzard hub repo create` and `edit` gain `--batch-size`, `--batch-wait`, and `--train-prefix`, and `--policy` admits
`forge-queue`; `blizzard hub repo probe <repository>` takes and prints a probe. `contracts/cli/hub.json` is regenerated.

## Admin screens

The repository form gains the batching settings, shown when the policy is one the hub lands itself, with a note that the
repository's CI must run on pushes under the train prefix, and offers `forge-queue` with its fallback named when the
last probe did not show a queue. The repository detail shows the last probe and a button that takes a new one.

## The board

A batching flow's lane (`web/projects/hub/src/app/board/flows/`) shows the batch in progress where a lone landing shows
its chunk: a group of the chunks it carries, titled with its state and how long it has held it — "proving 4 · 6m" — the
train branch linked. Chunks ready for the next batch sit in the line behind it. A chunk ejected in the last day shows at
the lane's end with its verdict and the batch it left, until the board window passes; a split shows as the half being
proven, labelled as split from the batch before. A `forge-queue` lane shows each chunk's place in the forge's queue
beside the order the flow received them, with the forge's queue linked.

A card's line under its node reads "in a batch of 4 on `blizzard/master`", "ready for the next batch on
`blizzard/master`", or "3rd in GitHub's merge queue on `blizzard/master`". The chunk detail's timeline shows each batch
its submission joined, how each ended, and its ejection, linked to the flow's record.
