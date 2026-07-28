# Plan — `epic:forge-status`

An operator who runs their backlog from a forge tracker gets no answer there to the first question they have: has the fleet taken this? The tracker shows open issues; the board shows chunks; the link between them lives in the operator's head, rebuilt every time they cross-reference the two. A teammate who never opens blizzard sees even less — to them the fleet is invisible, and an issue an agent is mid-way through looks exactly like one nobody has touched.

The fix is a projection, and the word is chosen carefully: the hub is the truth, and the forge carries a coarse, one-way reflection of it as labels — `blizzard:ingested` when the fleet holds an issue, `blizzard:in-progress` once work has begun, nothing once it no longer holds it. Coarse is the point. The forge answers exactly two questions — does blizzard have this, and has work started — and everything finer belongs to the board: a question waiting on a human, a paused chunk, an escalation. Answering "what needs me" from the tracker would teach people to triage the fleet from the wrong surface, so the label vocabulary deliberately cannot express it.

Reconciliation is the system, not the safety net. A periodic sweep asks the forge for everything wearing a `blizzard:*` label, derives the desired label for every work ref the hub still holds, and writes the difference — which makes removal nothing special, tolerates every missed event, and needs no memory of what was ever labeled. Event-driven immediacy can ride the hub's existing change feed later as a latency refinement; it earns nothing in correctness.

## What to build

- **The vocabulary and its derivation.** Two labels, exactly one (or none) per issue, derived per work ref from the live holding chunk: `ingested` for a chunk at rest, `in-progress` once anything has happened to it — running, paused, parked on a question alike. A stopped or done chunk, or no live holder at all, clears the label.
- **The annotate capability on the work-source seam.** An optional write half beside today's deliberately read-only fetch: set or clear an issue's status, list what is marked. The adapter owns the rendering — GitHub spells it `blizzard:ingested`, a Jira binding may not get colons — and owns making its labels exist before first use. A source without the capability, or a token without write scope, degrades to today's behavior rather than failing ingest.
- **The stateless reconciler.** A modest periodic sweep in the hub: query by label, diff against hub truth, write only differences. Best-effort by construction — a slow or absent forge never blocks a chunk transition, and no annotation state is stored hub-side.
- **The opt-in.** Per-work-source configuration, off by default. This workspace routinely runs snapshot and feature-env hubs against real data; only the canonical instance may write labels, or two writers fight over the same issues.

## Deliberately out

No `needs-human` label — the board is where the fleet asks for people. No issue comments — append-only surfaces cannot be reconciled the way labels can. No reading labels as input, no webhooks, no bidirectional sync: humans removing a `blizzard:*` label get it re-asserted, because the label is a display, not a control.

## Open questions

- Whether `blizzard:delivered` earns a slot. Delivery is hub-observable today (delivering status, PR-opened facts, landing markers), but a land PR that closes the issue already tells the tracker the story in its native tongue.
- Where the write half lives: grown onto the work-source protocol itself, or a sibling capability the registry exposes — the read seam's current shape looks intentional and deserves an architecture pass.
- Whether the projection buckets by derived status or by underlying facts. A chunk paused before its first run reads `in-progress` under status bucketing — defensible, slightly smelly, and cheap to revisit since the facts are independently readable.
