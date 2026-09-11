# Plan — `epic:cadence`

Four epics already promise the fleet a schedule — mutation testing's corpus pass, fact-egress's export, ideation's
routine, and the trends that gardening can only draw from repeated measurement — and each of them would otherwise build
a timer of its own. The fleet should keep its own schedule instead: a routine carries an interval, and when the interval
comes due the hub mints its work. Sunday at four, the architecture sweep enters the queue like any other work and waits
its turn.

## What to build

- **A routine that declares its own rhythm.** An interval, and a minimum spacing between runs, stated once where the
  routine is defined.
- **Minting at the hub when due.** The hub creates the work; the queue decides when it runs. A fleet behind on work
  falls behind on its own housekeeping rather than displacing an operator's.
- **Spacing measured from the last finished run.** A routine an operator ran by hand on Saturday does not run again on
  Sunday — the week it asked for is a floor, not a slot it must occupy.
- **The existing four, moved over.** The epics already promising a cadence declare one here instead of scheduling
  themselves, which is how the machinery earns its place rather than adding a fifth way to do this.

## Open questions

- What a missed window does: skip quietly, or catch up — and how many catch-up runs a fleet that was off for a week
  owes.
- Whether a routine is per project or per hub once `epic:projects` gives projects a home.
- Where an operator's manual run is recorded so that spacing can see it, since a hand-run pass that the scheduler does
  not know about defeats the whole point.
