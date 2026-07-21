# Plan — `epic:cost`, core slice

> **Shipped.** This plan is frozen: it records what cost controls were conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed. Model routing by cost remains live scope in the epic's registry history.

An overnight fleet spends money while its owner sleeps. Running the MVP for real made this the first lesson: token-spend runaway is a first-order operational hazard, and "the harness's own billing page" is not a ceiling, it is a receipt. The owner needs to decide in advance how much a night is allowed to cost — and trust that the fleet will stop itself at that line rather than explain itself in the morning.

## What to build

- **Spend as facts.** Token and cost telemetry recorded the way everything else is recorded — usage facts at the source, flowing through to the hub — so what a chunk, a node, or a night actually cost is a query, never an estimate.
- **Spend on the board.** The operator sees cost where they already watch the fleet: per chunk, and rolling up, live.
- **Budget caps.** Per-chunk and per-night ceilings the operator sets in advance. The fleet stops itself at the line — work parks rather than spends past it.
- **The kill-switch.** The hand on the cord when trust runs out: one act that halts spend fleet-wide, for the night the caps turn out to have been set wrong.
- **Model routing by cost** — sending work to cheaper models where fit allows — named as scope here, deliberately last: the caps and the telemetry must exist before routing against them can mean anything.
