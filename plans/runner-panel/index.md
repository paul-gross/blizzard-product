# Plan — `epic:runner-panel`

> **Shipped.** This plan is frozen: it records what the runner panel was conceived to be, and its entry in [delivered.md](../../delivered.md) is the record of what landed.

Some of the fleet's most important material can never appear on the board, by design: transcripts, leases, heartbeat freshness, resume commands all live on the runner's machine, and the hub never holds them — it cannot leak what it does not hold. But the person debugging a bad night needs exactly that material, next to the fleet state that explains it. The runner panel is the one surface that can show both: a web app served by the runner itself, read by a browser on that machine or its private network — a client *of the machine*, never something dialing into it.

## What to build

- **The machine panel.** The runner's local truth in a tab: its capacities with used and free, hub connectivity and buffered-event depth, leases with their derived state and heartbeat freshness, held environments, machine-local chunks, open asks — and every escalation rendered with its literal resume command, copyable.
- **The transcript viewer — the core pitch.** A chunk's session transcripts rendered in place, beside the lease facts and judgements they explain. The fact record says *what* happened; the transcript says *why* — and [`persona:harness-engineer`](../../charter/personas/harness-engineer.md)'s incident reconstruction needs them on one screen.
- **One codebase, two thin apps.** Fleet views exist once, in a shared library: the hub's build composes them alone and *is* the board; the runner's build composes the same libraries plus the local panel. The runner app can never fork a fleet page, and the hub build carries no panel code.
- **Local controls, and only local ones.** The panel's controls are exactly the runner's write surface — pause and start, requeue, selftest — plus takeover surfaced as the copyable command, because taking over is a terminal act and the browser only hands the human to the terminal. Fleet controls stay the board's: local facts get local controls, hub facts get hub controls, and the panel is merely the one tab showing both.
- **Degraded by construction.** With the hub unreachable, fleet pages banner and go stale while every local page and control keeps working. The machine panel is precisely the surface that must not depend on the hub.

## Artifacts

The panel's founding mockups live in [artifacts/](./artifacts/):

| Mockup | What it explores |
|--------|------------------|
| [runner-panel.html](./artifacts/runner-panel.html) | The machine panel: capacities, hub link, tick and lease state, escalations with copyable commands. |
| [runner-transcripts.html](./artifacts/runner-transcripts.html) | The transcript viewer: sessions on disk beside the store's derived state, with follow mode. |
