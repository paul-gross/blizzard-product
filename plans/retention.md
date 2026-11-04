---
epic: retention
refinement: scaffolded
status: horizon
---

# Plan — `epic:retention`

The hub keeps everything. Every fact, transcript, artifact, question, and measurement it has ever received is still
there, which is a comfortable arrangement for about a year and then arrives one morning as a daemon that will not start.
The operator never chose this; storage was simply assumed to be free. The honest version hands them the decision: each
kind of exhaust is kept for as long as they find it useful, and then let go. This is far cheaper to design now than
after there are years of facts to move.

## What to build

- **A declared policy per class of exhaust.** Facts, transcripts, artifacts, questions, and measurements age out on
  their own terms, because they are useful for very different lengths of time. The operator states each one; the
  platform stops guessing.
- **The derivation rule, enforced.** What leaves is only ever what nothing derives from. A record some live status is
  computed from is never a retention candidate, regardless of age — the fleet must not be able to forget its way into
  lying.
- **Expiry as a pass of the hub.** Letting go of old records is the hub's own housekeeping over its own rows, not work
  for a runner, so it runs as one of the hub's sweeps, on a timer of its own and on a floor that keeps it cheap, the way
  fact egress's export runs. A cadence mints work for runners and is not the place for it.
- **Export before drop.** `initiative:fact-egress` decides what has already left the building; nothing is dropped that
  the operator asked to keep and has not yet been handed.

## Open questions

- The granularity a policy is worth stating at: per class alone, or per class and project, and whether an operator
  thinks in ages or in sizes.
- Whether expiry is deletion or relegation to something colder and cheaper the hub no longer has to open.
- What a partially-retained chunk shows when someone opens it a year later — a detail page whose transcript is gone
  needs an answer better than a blank panel.
