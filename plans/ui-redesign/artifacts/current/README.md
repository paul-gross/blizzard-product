# Current: the fleet as a river over time

**Thesis.** A kanban board freezes something that is really motion. It shows where each chunk sits, but not how it got
there, how long it has waited, how often it went backwards, or what is queued behind it. Current puts **time on the
horizontal axis and the graph's nodes on the vertical**, and draws every chunk as one line through the nodes, like a
Marey train timetable. A chunk that dwells draws a long flat stretch. A bounce draws a loop that swings back up the
nodes. A chunk that lands drops a tick sized by its cost. A halted chunk draws a dashed vermilion tail that grows until
someone acts. Left of the **now** line is the past. The **present column** shows what sits in each node right now. To
the right is the **upstream**: ready and backlog chunks laid out by dependency depth, with the threads that block them.
Runner capacity runs along the top and the subscription windows along the bottom. Together they are **the banks**, and
when the banks are full the flow is throttled. Every view opens with one sentence that says what the picture shows, so a
glance is enough.

## Principles

1. **Position carries meaning before colour does.** Height is the node, horizontal position is time, line thickness is
   accrued cost, tick height is landed cost. Colour is held back for signal.
2. **Four signal colours, no decorative ones.** A warm near-black ground with bone ink and greys does the work. On top:
   **vermilion** for a human being needed (halts, pooling, dams, throttles), **orchid** for bounces, **lime** for the
   live head of a running chunk, and **bone** for landed work. Done work is quiet on purpose. A needs-human chunk is the
   loudest thing on screen and the 53 ready chunks are the quietest.
3. **Nothing moves unless state changed.** No spinners, no breathing glows, no marquee. A state change flashes once (a
   ring on the line head, a highlight on the chip). A new halt pulses three times and then stays still. The past creeps
   left as time passes but is never tweened.
4. **Say it in words.** The verdict line ("Pooling at deliver: 4 chunks, 2 waiting on you. 24 dammed behind C-6J9H.
   Capacity-bound.") is computed from the same model the picture draws. Serif italic is used only for these sentences,
   so the eye learns that serif means "this is the answer".
5. **Blast radius over counts.** The baseline showed "54 ready". Current shows that 24 of them sit behind one halted
   chunk. The upstream threads make that dam visible, and the needs-you card says "holds back 24".
6. **Cost is honest and accrues where it happens.** Lines thicken as cost accrues, landed ticks grow with cost, the
   spend curve is built from per-chunk accrual, and `~` marks estimates, as the baseline does.

## Anatomy

**Board, desktop** (`board-desktop.html`)

- *Verdict bar*: the sentence, landed today, median cycle, and the live clock.
- *Slots bank* (top): how many slots were held over time against the ceiling. Stretches at the ceiling are hatched
  vermilion ("saturated since 12:49"). winter's idle slots are named, not hidden.
- *Lanes*: Triage, Build, Review, Mutation, Pre-push, Deliver, Landed. The lane label shows how many chunks are in the
  lane, the median dwell, and one occupancy tick per chunk. A lane turns vermilion when something in it is halted, or
  has waited more than 2.5 times the lane's median.
- *Present column*: one chip per chunk in flight, ordered with anomalies first. Each chip shows id, title, dwell and
  cost. Triage, when empty, says what would enter next and why it can't ("12 ready, waiting for a slot").
- *Upstream*: ready chunks sit in columns by chain depth, grouped by epic. A thread is drawn to each chunk's critical
  blocker; the rest appear on hover. Threads that trace back to a halted chunk turn vermilion. The axis gives a rough
  ETA by depth ("+9h", "+28h").
- *Windows bank* (bottom): pace (spent divided by elapsed) for Anthropic 5h, Anthropic 7d and OpenAI 7d, with a
  projection past now. "r-chatgpt declining until 14:41" is drawn as a hatched span over that future.
- *Spend*: the cumulative spend curve since midnight, with yesterday's total as a ceiling.
- *Rail*: **Needs you** (reason, blast radius, bounces, cost, attempts, and one-key take over / requeue / PR), then
  **Focus** (the focused chunk's node history, with a dwell bar per visit, bounce reasons in orchid, tokens and cost),
  then **Live** (every state change).

**Fleet, desktop** (`fleet-desktop.html`) uses the same time axis, with the banks as the subject: the subscription
windows as pace lines with projections and roll times; every environment slot as a lane showing which chunk held it,
coloured by node, with bounces and halts marked and idle time hatched, so wasted capacity is visible; a "next" ghost in
free slots; and spend by runner. The rail inspects one runner: its state sentence, binds, harness, version, a capacity
map of all 14 environments, held chunks, and pause, drain and logs.

**Mobile** asks one question per screen. `board-mobile.html` shows the verdict, then **the river**: a vertical outline
of the graph whose width at each node is how much is sitting there. It widens and glows vermilion where work pools, and
bounce arcs run down its right side with counts. Below that are the needs-you cards with 40px Take over / Requeue / PR
buttons, then spend. `fleet-mobile.html` shows the verdict ("Capacity-bound."), the windows as pace gauges centred on
"on pace", and the runners with an environment grid and Pause / Drain.

## Interaction model

| Keyboard         | Board                                                                                             | Fleet                                    |
| ---------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `j` `k` / arrows | next / previous in-flight chunk                                                                   | next / previous environment              |
| `n`              | next anomaly (halts first)                                                                        |                                          |
| `u`              | walk upstream: focus a chunk this one blocks                                                      |                                          |
| `t` `r` `o`      | take over (copies the resume command), requeue, open PR                                           |                                          |
| `tab`            |                                                                                                   | next runner                              |
| `p` `d`          |                                                                                                   | pause / resume, drain the focused runner |
| `[` `]`          | time window: 1h, 3h, 7h, 24h, 48h                                                                 | same                                     |
| `space`          | freeze the live stream                                                                            | same                                     |
| `x`              | preview the board at 80 in flight                                                                 |                                          |
| `⌘K` or `/`      | palette: chunks by id, ref or title, plus commands (take over, requeue, pause, promote, zoom, go) |                                          |
| `g f` / `g b`    | go to fleet / board                                                                               |                                          |
| `?`              | all keys                                                                                          | all keys                                 |

**Pointer.** Hovering a line, chip or upstream dot lights up that chunk's whole path, plus everything it blocks and
everything blocking it. Everything else dims, and the Focus panel previews the chunk without losing the pinned one.
Click to pin. On fleet, hovering a slot segment lights that chunk across slots.

**Touch.** Tap a node in the river to open a bottom sheet of the chunks there (dwell, cost). Needs-you actions are
full-width buttons, and Take over shows the resume command inline. Runner Pause is a real toggle that the simulation
honours.

**Live.** All four mocks run a simulation over the real chunk data, one simulated minute every 2s. Chunks advance
through their graph, review fails and land failures bounce back, chunks land and free slots, ready chunks with no unmet
blockers claim the free slots, and dependents unblock. r-chatgpt holds its slots until 14:41 because it is ahead of
pace. Requeueing C-6J9H releases the dam. Test hooks for headless runs: `#t=60` pre-runs 60 ticks, `#keys=j,n,x` replays
keys, `#hover=ZEVE` hovers a chunk, `#still` stops the clock, and `#sheet=5` opens a mobile sheet.

## From 8 running to 80

- **Lines become a density field.** In-flight history is drawn at low alpha and only the current dwell is bright, so 80
  lines read as a texture of where work flows, and the vermilion and orchid strokes still stand out. Press `x` to see
  it.
- **The present column collapses per lane.** Each lane shows its three most anomalous chips, then "+27 more · oldest
  31m". Occupancy ticks cap at ten and then show a count.
- **The banks are unchanged at scale.** Saturation is a ratio, so it reads the same at 10 slots or 80.
- **Upstream** wraps stacks beyond four per depth column, and the columns compress. Past roughly 150 ready chunks it
  needs an epic-level rollup (one dot per epic, sized by count). That rollup is designed but not mocked.
- **Mobile** never shows individual lines. The river's width is a count, so it scales without change.

## What it deliberately drops from the baseline

- **The KPI strip.** The nine counters are replaced by one sentence, and by the picture itself.
- **Status columns as the main layout**, including the "done max(48h, 10)" column. Done work is a tick on the Landed
  lane, sized by cost, and its detail is on hover.
- **The 53 identical ready cards.** Ready work is now dots in a dependency field, and its titles appear on hover or in
  the palette.
- **"Blocked by C-XXXX (ready)" text on every card.** It is drawn as a thread instead.
- **The 10-segment usage meters.** These are replaced by pace (spent ÷ elapsed) against a 1.0 line, which explains a
  decline and isn't just a fill level.
- **The host list and detail page.** With one host it fits in a single line of facts. With many hosts, the fleet lanes
  group by host.
- **All-caps letter-spaced monospace labels.** Sans is used for words, monospace only for ids and numbers, serif only
  for verdicts.
- **The open questions and open gates panels when they are empty.** They become "asks 0 · gates 0" until something
  arrives, and then they join Needs you.

## Risks

- **Marey charts are unfamiliar.** Most operators will need a minute to learn that "up is backwards". The loops and the
  verdict sentence help, but onboarding needs a one-time legend overlay.
- **The vertical node axis is a union across graphs.** Graphs with different nodes (survey, mutation-less bas-dwf) share
  lanes, and a graph with a node outside the union needs a lane policy.
- **The upstream ETA axis is a guess.** It multiplies depth by median cycle time. It is labelled with `~` and `+h`, but
  it could still be read as a promise.
- **Pace history is synthesised in the mocks.** The real hub samples usage windows sparsely (every 44s to 4m), so the
  pace line will be steppier than drawn.
- **Dense SVG redraws.** The mock redraws the whole field on every tick. At real scale this needs canvas, or a retained
  scene with per-chunk updates.
- **Colour reliance.** Vermilion vs orchid must survive colour-blind palettes. Shape helps (halt bar, dashed tail, loop
  curve), but it should be tested.

## Type and palette

Noto Sans at semi-condensed width for words, Noto Sans Mono with tabular figures for ids, times and money, and Noto
Serif italic for verdicts only. These are local system fonts, so the mocks render offline. Colours: ground `#12110e`,
ink `#ede7da`, signal vermilion `#ff5a35`, orchid `#ee82d4`, lime `#c6f03c`.

## Files

- `board-desktop.html`, `fleet-desktop.html` (1600×1000), `board-mobile.html`, `fleet-mobile.html` (390×844):
  self-contained, inline CSS and JS, no network.
- `board-desktop.png`, `fleet-desktop.png`, `board-mobile.png`, `fleet-mobile.png`: headless Chrome renders of the
  opening state.
- `board-desktop-hover.png`, `board-desktop-live.png`, `board-desktop-80.png`, `board-mobile-sheet.png`: the same mocks
  after hovering a dammed chunk, after 60 simulated minutes, in the 80-in-flight preview, and with a mobile sheet open.
