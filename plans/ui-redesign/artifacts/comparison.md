# The eight proposals compared

Five proposals are experiments, and one is the control: a conventional SaaS application built with care, against which
the experiments' gains can be measured. Two more are classic variations on the control, each drawn from a different
family of SaaS product and each borrowing the experiments' strongest elements. Each proposal answers the [brief](./brief.md) differently, and each one starts
from a different question about what the board is for.

| Proposal                           | The board's question               | Direction   |
| ---------------------------------- | ---------------------------------- | ----------- |
| [Editorial](./editorial/README.md) | What is the headline?              | Modern      |
| [Altitude](./altitude/README.md)   | How close do you want to look?     | Modern      |
| [Current](./current/README.md)     | Where is the work pooling?         | Modern      |
| [Synop](./synop/README.md)         | What is the weather over the work? | Avant-garde |
| [Tacit](./tacit/README.md)         | What will the fleet do next?       | Avant-garde |
| [Console](./console/README.md)     | What is open, and in what order?   | Classic     |
| [Desk](./desk/README.md)           | What is waiting on me?             | Classic     |
| [Dispatch](./dispatch/README.md)   | What has stopped, and when lands?  | Classic     |

## Each one's strength and its risk

- **Editorial** uses typographic hierarchy as the whole information architecture. The two chunks that need a human are
  set as lead headlines, and the 53 ready chunks shrink to agate texture. It is the least to learn of the five. Its risk
  is that it depends on a variable font with width, weight and optical-size axes, and its agate may be too faint at low
  DPI.
- **Altitude** draws one continuous space at five fidelities: field, glyph, row, panel and transcript line. Board and
  fleet are two projections of the same objects, driven by a composable query line (`needs:human | sort holds`). It
  offers the highest ceiling for mastery. Its risks are that the field reflows as state changes, so it may not hold a
  stable map, and that its fold-calm-chunks thresholds could hide a chunk that is quietly stuck.
- **Current** draws a timetable chart of the fleet: time on one axis, nodes on the other, bounces as loops. Ready work
  sits upstream, joined by blocking threads, and capacity and subscription pace are the banks. It is the only proposal
  that shows *where* the fleet struggles: the tangle of loops at deliver and pre-push. Its risks are that "up means
  backwards" takes learning, and that the upstream ETAs read as promises.
- **Synop** is a weather chart. Each epic is fixed land, each chunk a station plot, and blockers are fronts.
  Subscriptions are air masses that saturate into fog. Of all five it carries the most information per pixel, and it
  makes blocking chains and throttling physically visible. Its risks are that it is the steepest to learn, that titles
  leave the board, and that its geography must either reshuffle or fill with dead land as epics come and go.
- **Tacit** shows motions instead of state. Each thing the fleet will do carries a fuse, silence is consent, and
  standing orders absorb repetition, so the screen empties as the operator masters it. It holds the clearest answer to
  "does the fleet need me?" and the best mobile contract. Its risk is that it stands or falls on forecast accuracy: once
  the fuses prove wrong, the operator stops trusting silence and wants raw state back.

- **Console** is the classic SaaS control. It has a collapsible left nav with an Inbox badge, a grouped chunk table in
  attention order with the needs-human group pinned and actionable in the row, saved views, a ⌘K palette, a push drawer,
  and a compact mode that is a single density switch. It costs the least to learn, and its pinned needs-human group sets
  the bar each experiment must beat. Its risks are that it reads as one more Linear-style app, and that a table-first
  board loses the kanban's sense of flow.
- **Desk** takes the shared support inbox (Front, Intercom, Superhuman). Halts, asks and gates form one queue with
  response-time clocks, assignment, snooze and macros, the node history is the conversation thread, and a note can travel
  with a requeue as guidance for the next agent. Inbox zero is where the operator walks away, so that screen carries
  Tacit's forecast of what the fleet will do without them. Its risks are that it has no flow view at all, and that notes
  travelling with a macro need blizzard to support them.
- **Dispatch** takes fleet and dispatch consoles (Samsara, Onfleet, Flexport) with a Stripe-style spend view. Exceptions
  are pinned first, and a conventional Gantt of slots over time carries Current's bounce loops, with the past left of the
  now line and forecasts right of it. Altitude's query line is its search box and Synop's blocking chains become a
  dependency diagram. It is the only classic proposal that answers "when". Its risks are that its ETAs may read as
  promises, and that the timeline needs width a laptop may not have.

## What the five experiments agree on

The five experimental designers worked independently from the same brief. Where they converged anyway, the agreement is
either a finding about the operator or a shared model default, and the two need telling apart.

- **Rank needs-human chunks by what they hold.** Four proposals lead each one with the work it blocks. They disagree on
  the count, though: C-6J9H holds four, 23 or 24, depending on whether only direct dependents count or the whole chain
  does. A build has to settle that definition, because it decides the order of the needs-you items.
- **A computed sentence replaces the KPI strip.** All five open with prose: Editorial's lede, Current's verdict, Tacit's
  posture sentence, Synop's synopsis and Altitude's section heads. This is likely a real finding. The operator's first
  read is a judgement, not a set of counts.
- **Colour means one thing per hue, and vermilion means a human is needed.** All five use vermilion for needs-human.
  Four of the five also chose ink on warm paper, and Current chose warm near-black. That palette is very probably the
  current model default rather than a choice any of them made, so challenge it before building on it.

## What could be combined

The proposals are not exclusive. Altitude's query line is an input model and would work over any of the other surfaces.
Tacit's "if you walk away now" forecast and its standing orders could carry Editorial's lede. Current's flow chart could
be an altitude in Altitude. Synop's fronts are a way to draw blocking chains that any proposal could borrow.
