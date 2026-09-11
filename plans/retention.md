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
- **The sweep as ordinary work.** Expiry runs on a schedule rather than on a boot, riding the cadence machinery
  `epic:cadence` builds instead of growing a timer of its own.
- **Export before drop.** `epic:fact-egress` decides what has already left the building; nothing is dropped that the
  operator asked to keep and has not yet been handed.

## Open questions

- The granularity a policy is worth stating at: per class alone, or per class and project, and whether an operator
  thinks in ages or in sizes.
- Whether expiry is deletion or relegation to something colder and cheaper the hub no longer has to open.
- What a partially-retained chunk shows when someone opens it a year later — a detail page whose transcript is gone
  needs an answer better than a blank panel.
