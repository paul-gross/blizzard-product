# Dispatch: the board as a fleet operations console

**Thesis.** Blizzard runs a fleet, so its board should borrow from the products people already use to run fleets:
Samsara, Onfleet, Flexport, and airline and rail operations consoles. In those products the first screen answers three
questions in a fixed order. What has stopped and needs me? What is moving, where, and when will it arrive? Do I have
capacity, and what is it costing? Dispatch answers the same three for blizzard. Stopped chunks sit first in an
**exceptions queue** with one-click resolution. The middle of the page is a **dispatch board**: a timeline in which
each row is a runner's environment slot, each bar is a chunk moving through its stops, a bounce draws as a return arc,
and everything right of the now line is forecast. That forecast covers each chunk's ETA, the next load each slot will
take, and the stretch in which a throttled runner takes no new work. Capacity and subscription windows sit beside it,
treated as **hours of service**. Spend gets a Stripe-style ledger with `~` on every estimate. The chrome stays the
neutral, classic SaaS that Console set as the bar. What changes is the organizing idea. Console asks "what is open, and
in what order?" Dispatch asks "what is stuck, what is moving, and do I have the capacity?", and it draws time so that
the answer includes *when*.

## Principles

1. **Exceptions first, and actionable where they appear.** A stopped chunk is an exception. It is pinned above
   everything, ranked by what it holds, never capped, and carries **Take over**, **Requeue** and its PR on the card. It
   is the only red on the page.
2. **Time is an axis, not a column.** Where a chunk *is* matters less than how long it has been there, how often it
   went back, and when it will land. The dispatch board puts the past on the left of the now line and the forecast on
   the right. A dwell is a long bar, a late stop turns amber, and a halt is a red hatched bar that grows until someone
   acts.
3. **Forecasts are drawn as forecasts.** Anything that has not happened yet is a dashed outline marked with `~`: ETAs,
   next loads, expected pickup times. Measured things are solid. The two never share a style.
4. **Capacity and hours of service are first-class.** Slots are capacity. Subscription windows are duty clocks, so
   "declining until 14:41" is drawn as a hatched band across that runner's future, not buried in a status string.
5. **Hue only means status.** The chrome is cool slate greys with a graphite sidebar, and the primary button is ink.
   Red means a human is needed. Amber means waiting, throttled, late or bounced. Blue means running, teal delivering,
   green delivered or under budget. Spend charts are drawn in ink, not a brand colour.
6. **Say it in a sentence.** Every page opens with one computed sentence in place of a KPI strip: "2 shipments are
   stopped and need you — C-6J9H holds 5 behind it. 8 moving on 10 of 14 slots; r-chatgpt takes no new loads until
   14:41 and winter's 4 slots sit idle. Next delivery ~13:16 (C-27A6). Spent ~$114.44 today, ~$2.69 under yesterday's
   pace." The ids in it are links.
7. **Honest numbers.** `~` marks estimates, as in the baseline. "Holds" counts the transitive closure over blocking
   edges the hub knows by id, with direct dependents shown beside it ("holds 5 downstream · 2 direct"). A blocker the
   hub does not name is labelled "unnamed", not guessed.

## Borrowed vocabulary, kept nouns

Dispatch borrows the *conventions* of logistics software, not its words. Operators already know chunk, runner, slot,
node and bounce, so the UI keeps them. The metaphor shows up in layout and interaction:

| Logistics convention              | Blizzard concept                 | Where it appears                                              |
| --------------------------------- | -------------------------------- | ------------------------------------------------------------- |
| Exception that stops a shipment   | needs-human chunk                | Exceptions queue, nav badge, mobile tab badge                 |
| Dispatch board / Gantt of drivers | environment slots over time      | The dispatch board                                            |
| Shipment tracking timeline        | node history, then forecast      | The **Tracking** tab of the chunk drawer and mobile sheet     |
| Returns                           | bounces                          | Return arcs on bars, the ↺ marker, "spent on returns"         |
| Vehicle capacity                  | slots per runner and workspace   | Capacity card, vehicle cards on Fleet                         |
| Hours of service / duty clock     | subscription windows (5h, 7d)    | Hours-of-service card, duty-clock rings on Fleet              |
| Loading dock, next out            | ready queue in claim order       | Loading dock panel: next out with expected pickup, held chains |
| ETA                               | median dwell per remaining node  | Forecast bars, drawer header, moving rows                     |

Two words are new on screen. **Exceptions** names the needs-human queue, which is what Console called Inbox. **Stopped**
replaces "needs_human" as the verb on a card, because "stopped 22m" reads at a glance in a way a status code does not.
"Shipments" is the name of the table layout. The rows in it are still chunks.

## Information architecture

```
[blizzard mark · paul-gross / blizzard hub · hosted]   ← tenant
[Search or jump to…  ⌘K]
Operate
  Dispatch                      ← the board: exceptions, dispatch board, loading dock, rail
  Exceptions          (2) red   ← the board filtered to is:exception (asks and gates join here)
  Shipments            64       ← the board's table layout (V toggles)
  Fleet                 ● amber ← runners as vehicles, hours of service, slots, host
  Spend              ~$114      ← Fleet's Spend tab: the ledger
  Events
Workflow
  Graphs · Gardening
Projects                        ← the project lens, global
  All projects 64 ✓ · ■ blizzard 64 · ■ winter 0
                     (spacer)
Settings & admin
[PG Paul Gross · Owner]
```

- The sidebar is **graphite even in the light theme**. Fleet and logistics consoles use a dark rail, and it separates
  navigation from the work surface without spending a hue. It stays neutral, so the only colour in it is the red
  Exceptions badge and the amber dot on Fleet.
- **Spend has its own nav entry.** Cost is the third question an operator asks, so it gets a place of its own rather
  than a corner of Usage.
- The top bar keeps the baseline's facts one glance away: `Live · hub ok | Today ~$114.44 · yday ~$161.25`. The status
  bar on every page carries slots, moving count, the three windows (OpenAI 7d in amber with "declining → 14:41"), the
  last event and the clock.

## The Dispatch page

At 1600×1000 the page reads top to bottom in the order an ops console uses.

**Summary sentence and toolbar.** The sentence comes first. Under it, saved views sit as a segmented control (Open 64,
Exceptions 2, Moving 8, Dock 53, Delivered 38; keys `1`–`5`), beside the **query line**. The query line is Altitude's
composable filter, adapted into a classic search box. It takes `is:held epic:STRICT sort:holds`, with keys
`is node runner epic holds> cost> bounces> sort` and bare words. It highlights tokens in place, flags unknown values
with a wavy underline, and completes with Tab. Saved views are just stored queries, and the query line shows which one
is active. On the timeline, a query dims the lanes and chain nodes it does not match. On the table, it filters.

**Exceptions.** Two cards, side by side, with a red left edge. Each one shows the id, epic and title, the reason in
the hub's words ("Bounce cap crossed after 6 bounces at deliver · takeover"), and a meta line: stopped 22m (live), at
deliver on r-claude, holds 5 downstream, 6 bounces, $8.20. On the right are **Take over** (ink, `T`), **Requeue**
(`R`, confirms), and the PR. Ranking is by holds, then age.

**The dispatch board.** Rows are environment slots, grouped by runner and workspace: r-claude on blizzard-claude 1–6,
r-chatgpt on blizzard-chatgpt 1–4, and winter collapsed to one row reading "Idle all window · no winter work is
queued". The label column names what each slot holds now (glyph, id, current stop, or "stopped" in red).

- **Bars.** A chunk in flight is one bar split into stop segments: triage, build, review and so on, labelled where they
  fit. The live segment takes its status colour: running blue, delivering teal, late amber (C-2475 reads "build · 1h
  41m · late", three times the median), and halted red with a hatch that grows. Earlier loads in the same slot are
  muted bars with a tick: "C-VA7X delivered 12:24".
- **Bounces** are amber return arcs from where a visit failed back to the start of the stop it returned to, plus an
  amber underline on the failed segment. C-6J9H's five deliver failures draw a row of small loops at the end of its
  bar. You can see where the fleet is struggling without reading a single event.
- **Forecast.** The area right of now is tinted. Each moving chunk gets a dashed bar to its ETA ("ETA ~13:52"),
  followed by a dashed ghost of the next load the dispatcher will put in that slot ("next C-DF68"). A stopped chunk
  gets a red dashed "awaits you" with no ETA. Across r-chatgpt's four lanes an amber hatched band runs from now to
  14:41: hours of service, during which it takes no new loads. Its next loads start at 14:41.
- **Time window.** 2h, 4h (default), 8h and 12h, set by the segmented control or `[` and `]`.
- **Hover** lights the chunk wherever it appears (its bar, any earlier claim in another slot, its node in the chains)
  and shows a tooltip with the segment's stop, start, end and dwell, plus ETA, cost, bounces, holds and what it waits
  on. Click opens the drawer.

**The right rail.** Three cards:

- **Capacity:** "10 / 14 slots · 71% utilized", a per-runner slot bar coloured by status, a utilization sparkline since
  06:00 against blizzard's cap of 10, and a sentence on why: blizzard full since 12:51, winter idle all day.
- **Hours of service:** one meter per window. Fill is spend, the ink tick is elapsed, and fill past the tick means
  over pace. Each meter has its reset time and pace (0.17×, 1.00×), and OpenAI 7d shows "declining → 14:41" in amber.
- **Spend today:** ~$114.44 large, "~$2.69 under yesterday at this hour", a cumulative line for today against a dashed
  yesterday, and the three largest loads delivered today.

**The loading dock.** This sits below the board and is the ready queue seen as a dock.

- **Next out** lists the 12 unblocked chunks in claim order, each with an expected pickup time worked out from when
  slots free up. Pickups on r-chatgpt slots wait for 14:41.
- **Held** draws the blocking chains as dependency links, which is Synop's idea of making blockers visible, rendered as
  a conventional left-to-right dependency diagram. C-6J9H is a red root with a dashed red link to C-ZEVE → C-MG3R →
  C-4J2W → C-4HAX, and a branch to C-N1JR. In-flight blockers (C-PTP2, C-2475, C-MFJE) are tinted roots. A chunk held
  by blockers the hub does not name starts from a dashed "3 unnamed" root. The 19 held chunks with no known edges are
  listed underneath, not drawn as fake chains.

## The Shipments layout

`V` swaps the timeline for a grouped table, Console's strength, kept as a peer. The groups are:

- **Exceptions** (tinted, pinned, two-line rows with the reason and a Take over button).
- **Moving**, soonest ETA first, with a six-segment stop track and a ↺ bounce count.
- **At the dock**, unblocked first, with "Show 41 held behind blockers" as a toggle.
- **Backlog**.
- **Delivered · 48h**, collapsed.

The columns are glyph, ID, shipment (epic, title, ref, +N work items), stops, runner · slot, waits on · holds, ETA,
cost and since. Adding a `sort:` token flattens the groups into one ranked list, so `is:held sort:holds` shows what
would unblock the most.

## The chunk drawer: tracking

The drawer is 560 px and slides over the page. Its header carries the status pill, id, previous and next (`K`/`J`),
copy and close. Below that come the ref, epic, title, runner · slot, graph and attempts, then the actions (Take over,
Requeue, PR, ⋯), then a four-cell strip: Stop · Stopped for / At stop for · ETA · Cost. It has three tabs:

- **Tracking** is a parcel-tracking timeline. Each past stop shows a check, its time range, dwell and cost. A return
  shows an amber ↺ with "Returned to pre-push — land answered failure", and a run of identical returns folds into one
  row ("deliver · returned ×5, 12:23–12:46"). The current stop has its live dwell. A halt shows red with "Needs you".
  The remaining stops are dashed, each with an expected time, ending at "Delivered · ETA ~13:34". For a needs-human
  chunk the hub's verbatim reason sits above the timeline. Below it is the **blocking chain**: what this chunk waits
  on, as chips, and the tree of chunks it holds, each clickable.
- **Cost** is a per-stop ledger with tokens, a share bar, a total, and "spent on returns" in amber. C-WSCR's figures
  are the baseline's own, so it shows $0.30, 19% of its $1.58, on returns. Other chunks' splits are apportioned by
  dwell and marked `~`.
- **Details** lists the properties, token usage, and the work items.

## Fleet

The page opens with "1 host and 2 runners, 10 of 14 slots loaded: blizzard is at capacity and winter's 4 slots sit
idle. r-chatgpt takes no new loads until 14:41 — its OpenAI 7d window is 8% spent against 7% elapsed. xps last reported
34s ago." Its tabs are Overview, Spend, Runners, Hosts, Enrollment tokens and Audit log.

- **Vehicle cards**, one per runner. The header has the name and id, a state pill (Working / Declining until 14:41 /
  Paused), "2 need you", a live last-seen, **Pause** (`P`) and details. A declining runner gets an amber banner that
  explains the decline in one line. The body is split in two. **Loads** shows each slot as a tile (chunk, stop;
  stopped ones red), plus winter's free slots. **Hours of service** shows a duty-clock ring per window: the outer thin
  ring is elapsed, the inner thick arc is spent, amber when ahead of pace. Beside each ring are pace, reset time and
  headroom. The footer lists harness, agents, delivered today with spend, and version.
- **Utilization** shows every slot since 09:00 as a thin occupancy track coloured by status, with winter hatched as
  idle. A sentence explains the gaps: nothing was claimable before the survey runs became ready at 12:44.
- **Host card (xps).** Live last report, URL, token, agents, registration. Enroll, Revoke token (confirm), and
  **Retire host**, which is destructive and requires typing `xps`.
- **Workspaces & slots.** One row per workspace, one tile per slot, showing chunk, stop and live dwell. Free slots are
  dashed. Clicking a tile opens that chunk's drawer on the board.
- **Runner drawer.** The state explained in a callout, duty clocks, held loads, binds, version, and the actions Pause,
  Drain, Edit bindings and Retire (type the name).

## Spend

Spend is a Stripe- or Ramp-style financial page. Its sentence reads: "~$114.44 spent today, ~$2.69 under yesterday at
this hour. r-claude carried 75% of it on Anthropic. ~$3.32 of in-flight spend went to returns, most of it C-6J9H
looping at deliver."

- **Spend chart.** Large figures for today so far, the delta against yesterday at this time, yesterday's total, tokens,
  and cost per delivery. Below them, a cumulative-by-hour line for today drawn in ink, with yesterday dashed.
- **By runner** (r-claude ~$86.12 Anthropic, r-chatgpt ~$28.32 OpenAI, winter $0). **In flight**: ~$24.34 across 10
  chunks, split into billed and estimated-not-yet-billed, plus the cost of returns.
- **By day**, **by stop** (where in-flight money went: build, mutation, review…), and **costliest returns** (bounce
  cost per chunk).
- **Ledger.** Every chunk that spent today, sorted by cost, filterable with its own query line. The footer explains `~`
  and notes that lifetime cost sums past the day's total.

## Mobile

On mobile the operator is away from the desk, asking "does the fleet need me?".

- **Bottom tab bar:** Exceptions (red badge), Dispatch, Fleet (amber dot), Spend, More. The badge never hides behind a
  menu.
- **Dispatch:** the sentence, then exception cards with full-width **Take over** / **Requeue** / PR (42 px targets),
  then Moving rows (stop track, node, runner, cost, ETA), then a capacity strip with r-chatgpt's decline, then three
  next-out chunks and "Show all 53".
- **Exceptions tab:** the two stopped chunks, the empty asks and gates panels stated plainly, and a "Watching" list of
  late loads that are not exceptions yet.
- **Detail** opens as a bottom sheet with the four-cell strip, the Tracking / Cost / Details tabs, and a sticky action
  footer.
- **Fleet:** vehicle cards with slot bars, load chips, duty-clock rings, and Pause / Details. Then a slot map and the
  host. **Retire** is an iOS action sheet that requires typing `xps`.
- **Spend:** the cumulative chart, by runner, and the top of the ledger.

## Compact mode

Compact mode is one attribute, `html[data-density]`, read through tokens. Toggle it with `⇧D`, the top-bar button, the
palette, or More on mobile. It persists as `dz.density`.

| Token        | Comfortable | Compact |
| ------------ | ----------- | ------- |
| Timeline row | 30 px       | 24 px   |
| Table row    | 36 px       | 28 px   |
| Body text    | 13 px       | 12 px   |
| Controls     | 30 px       | 26 px   |
| Page gutter  | 24 px       | 16 px   |
| Card padding | 16 px       | 10 px   |
| Top bar      | 48 px       | 40 px   |

Compact also drops the exception cards' reason line (it stays in the drawer), shrinks timeline bars to 12 px, hides
slot-tile detail on Fleet, and on mobile hides the second line of moving rows. In compact mode the loading dock starts
above the fold, under the board. See `board-desktop-compact.png`.

## Design tokens

**Type.** Inter (variable, inlined as woff2, OFL) for everything. A system monospace stack is used for ids, refs and
versions only. All numbers use tabular figures. Weights are 400, 500 and 600, with no all caps. Sizes: 10.5 (epic
tags, bar labels), 11 (captions, axis), 12 (meta), 13 (body), 14 (summary sentence), 16 (drawer title), 20 (page
title), 26 (big figures).

**Colour roles.** Light is the default and dark uses the same role names.

| Role    | Glyph              | fg / bg (light)       | Means                                           |
| ------- | ------------------ | --------------------- | ----------------------------------------------- |
| `nh`    | solid disc + `!`   | `#c4232c` / `#fdeced` | A human is needed. The only red on screen       |
| `wait`  | ring + clock hand  | `#925400` / `#fdf2dc` | Waiting, throttled, late, bounce / return       |
| `run`   | half-filled ring   | `#1659c7` / `#eaf2fe` | Running at a stop                               |
| `dlv`   | ¾-filled ring      | `#0b7066` / `#e2f4f1` | Delivering                                      |
| `done`  | solid disc + check | `#1a7040` / `#e6f4eb` | Delivered; healthy; under pace                  |
| `ready` | empty ring         | `#4a5260` / `#eef1f4` | Ready / at the dock (deliberately grey)         |
| `focus` | 2 px ring          | `#2563eb`             | Keyboard focus and hover link. Never a status   |

Neutrals are cool slate: canvas `#f3f5f8`, surface `#fff`, lines `#e1e6ec` / `#ccd3dc`, text `#111722` / `#434c5a` /
`#626b79`, ink buttons `#111722`, and sidebar graphite `#161b22`. Dark mode uses canvas `#0d1015` and surface `#151a21`.
This is a cool palette on purpose. It is not the ink-on-warm-paper default that comparison.md flags.

Status is never carried by colour alone. The glyph shape, the label ("stopped", "late", "declining"), the hatch on a
halt or throttle, the dashed outline of a forecast, and position (exceptions pinned first) all carry it too.

## Interaction model

**Keyboard.**

- Anywhere: `⌘K` / `Ctrl+K` palette, `/` query line, `G` then `D` / `S` / `F` / `$` to go to dispatch, shipments,
  fleet or spend, `⇧D` compact, `⇧T` theme, `?` shortcut sheet. `Esc` always closes the topmost layer first.
- Dispatch: `J` / `K` (and arrows) walk a cursor through the exceptions and then the slot lanes, or the table rows.
  `↵` or `O` opens the drawer. `V` switches timeline and table. `[` / `]` change the time window. `N` jumps to the next
  exception. `1`–`5` apply saved views.
- On a chunk (cursor or drawer): `T` takes over (copies `blizzard runner takeover c-6j9h`), `R` requeues (confirm),
  `P` opens the PR, `C` copies the id. In the drawer, `J` / `K` step to the next and previous chunk.
- Fleet: `J` / `K` move between runners, `↵` opens the runner, `P` pauses or resumes, `1` / `2` switch Overview and
  Spend.

**The palette.** It opens with *Suggested* actions: "Take over C-6J9H — holds 5", "Take over C-QWYS", and the two
requeues. It searches every chunk by id, ref, title or epic, so `6j9h` and `#1449` both work. It runs navigation,
views, fleet actions (pause a runner, retire host) and preferences, and shows the shortcut for each.

**The query line** completes saved views when empty, keys as you type, and values after `:`. It also offers chunk ids
for bare words, and choosing one opens the chunk.

**Pointer.** Hover a bar, chain node or next-out row to light the chunk everywhere and get its tooltip. Click to open.
Clicking an id in the summary sentence, a chain node, a slot tile or a load tile opens that chunk.

**Touch.** 42–44 px targets on exception actions and sheet footers. Bottom sheets for detail. Action sheets for
requeue and every destructive action, with type-to-confirm for retire.

**Live.** The clock ticks every second, and stop dwells, "stopped 22m", last-seen and last-report ages, and today's
spend count up. The timeline redraws each minute, so the now line moves right and live segments grow. Every few
seconds a real-shaped event replays into the status bar, and the matching lane or row flashes once. Nothing pulses,
spins or animates without a state change.

## Scaling from 8 running to 80

This section describes the design at scale. None of it is mocked beyond the 8-running snapshot.

- **Exceptions are never capped.** At 15, the queue wraps to more rows of cards in a two-column grid, still ranked by
  holds, and the badge says 15. Past six, the cards drop to compact one-line rows with their actions inline.
- **The dispatch board at 80 slots.** Lanes group by runner and collapse per group. A collapsed group becomes one
  occupancy row: stacked status colour per minute, with exception and late markers still drawn. A group expands on
  click or when it holds an exception. Compact rows (24 px) fit about 26 lanes on one screen, and `Group:` can switch
  to workspace or current stop. Return arcs stay readable because they only ever span one lane. A useful extra at
  scale is a "sort lanes by trouble" toggle (stopped, late, most bounces first). It is designed but not mocked.
- **Capacity reads as a ratio** ("62 / 80"), and the utilization sparkline is unchanged. Hours of service stays one
  meter per subscription window, whatever the runner count.
- **The loading dock at 500 ready.** Next out stays a short list of the next 12 pickups. Held chains collapse to one
  node per root with a count ("C-6J9H → 23 held") and expand on click. The query line (`is:held epic:STRICT
  sort:holds`) does the rest.
- **Fleet at 10 hosts and 30 runners.** Vehicle cards become a vehicles table sorted by attention: needs you,
  declining, paused, working. Each row has a slot bar and duty-clock rings inline, and the card is the drawer.
- **The palette and URL state** make any of these one keystroke or one link away.

## What it deliberately drops from the baseline

- **The eleven-counter KPI strip.** It becomes the computed sentence. Spend sits in the top bar and hub health in the
  live pill.
- **Status columns as the main layout.** The kanban's columns become the exceptions queue (needs human), the dispatch
  board (running and delivering), the loading dock (ready), and Delivered as a view. The table keeps a grouped list for
  anyone who wants one.
- **"Blocked by C-XXXX (ready)" on every card.** It is drawn as dependency links in the dock and listed in the drawer.
- **The ten-segment usage meters.** They are replaced by fill-against-elapsed meters and duty clocks that explain a
  decline.
- **The right rail's empty asks and gates panels.** They move to the Exceptions tab and say "no agent is waiting on an
  answer" only there.
- **The repeating activity feed.** It becomes the status bar's last event, row flashes, and Events.
- **One monospace face in all caps, and decorative colour.** Inter at three weights, mono for ids, and hue for status
  only.

## Risks and what I am least sure of

- **Forecasts invite trust they may not deserve.** ETAs, next loads and pickup times are median-dwell arithmetic. They
  are dashed and marked `~`, but an operator may still read "~13:34" as a promise. If they prove wrong often, the
  forecast half of the board becomes noise. A build should show forecast error once delivered ("landed 13:41, 7m
  late") so trust is earned.
- **The timeline costs width.** At 4h the bars are readable. A 2-minute bounce is 6 px, and stop labels drop out on
  short segments (the tooltip and drawer carry them). On a 1280 px laptop the board needs the 2h window or compact mode.
- **The metaphor could feel cute.** I kept the domain nouns and borrowed only conventions for that reason. "Stopped",
  "Exceptions", "Loading dock" and "Hours of service" are the four places the metaphor reaches the screen. Each one
  should be tested with the owner.
- **Two homes for the same chunk.** A running chunk appears as a lane on the board and a row in the table. A stopped
  chunk also appears as an exception card. Hover linking and the shared cursor help, but it is more redundancy than
  Console has.
- **"Holds" is settled here as transitive closure over named edges: C-6J9H holds 5 (2 direct).** That agrees with
  Console but not with the 23 or 24 that other proposals counted by including unnamed blockers. The hub should expose
  every blocking edge by id so the number can be complete.
- **Mock data.**
  - Ids, titles, refs, costs, statuses, runners, slots, window percentages and blocking edges are the baseline's.
  - C-WSCR's visits and per-step costs follow its node history exactly. C-6J9H from 12:19 and C-QWYS from 12:05 follow
    the event log. Earlier visits, and the slots that past chunks such as C-VA7X or C-AFTK occupied, are shaped from
    claim times and attempt counts.
  - Per-stop costs for chunks other than C-WSCR are apportioned by dwell (marked `~`).
  - C-QWYS's escalation reason is inferred, because the captures show only its status.
  - ETAs, median dwells, the next-load plan, utilization samples, hourly spend, the runner spend split, window reset
    times and r-chatgpt's 13:04 decline start are illustrative. So are the five older days in Spend by day, which are
    hatched and labelled as such.
  - Runners, Hosts, Enrollment tokens, Audit log, Events, Graphs, Gardening and Settings are stubs that explain
    themselves.

## Files

| File                               | What                                                                         |
| ---------------------------------- | ---------------------------------------------------------------------------- |
| `board-desktop.html` / `.png`      | Dispatch: exceptions, dispatch board (4h), loading dock, rail (1600×1000)    |
| `fleet-desktop.html` / `.png`      | Fleet overview: vehicle cards, utilization, host, slots (1600×1000)          |
| `board-mobile.html` / `.png`       | Dispatch on mobile (390×844 @2x)                                             |
| `fleet-mobile.html` / `.png`       | Fleet on mobile (390×844 @2x)                                                |
| `board-desktop-shipments.png`      | The Shipments table layout                                                   |
| `board-desktop-drawer.png`         | Drawer on C-6J9H: reason, tracking timeline with folded returns              |
| `board-desktop-dark-cost.png`      | Dark theme, drawer on C-WSCR's per-stop cost ledger                          |
| `board-desktop-dark.png`           | Dark theme                                                                   |
| `board-desktop-compact.png`        | Compact mode                                                                 |
| `board-desktop-8h.png`             | 8h window with the keyboard cursor on a lane                                 |
| `board-desktop-hover.png`          | Hover on C-WSCR: linked highlight and segment tooltip                        |
| `board-desktop-dock.png`           | Scrolled to the loading dock: next out and held chains                       |
| `board-desktop-query.png`          | Query line `is:held sort:holds` with completion, table flattened and ranked  |
| `board-desktop-palette.png`        | ⌘K palette with suggested takeovers                                          |
| `fleet-desktop-spend.png`          | Spend: cumulative chart, by runner, in flight, by day, by stop, returns      |
| `fleet-desktop-drawer.png`         | Runner drawer on r-chatgpt (declining, hours of service)                     |
| `fleet-desktop-retire-confirm.png` | Type-to-confirm on Retire host                                               |
| `fleet-desktop-dark-compact.png`   | Fleet in dark and compact                                                    |
| `board-mobile-sheet.png`           | Bottom sheet for C-6J9H with tracking                                        |
| `board-mobile-exceptions.png`      | The Exceptions tab                                                           |
| `board-mobile-compact.png`         | Mobile in compact mode                                                       |
| `board-mobile-dark.png`            | Mobile in dark                                                               |
| `fleet-mobile-spend.png`           | The Spend tab on mobile                                                      |
| `fleet-mobile-retire-confirm.png`  | Retire action sheet with type-to-confirm                                     |

Every mock is a single offline file (inline CSS and JS, Inter inlined, no network) and accepts URL state for review:

- `?theme=dark`, `?density=compact`
- `?still=1` freezes the clock at the 13:10:04 snapshot (all PNGs use it)
- Board: `?view=shipments`, `?window=2h|4h|8h|12h`, `?q=is:held%20sort:holds`, `?drawer=C-6J9H`,
  `?tab=track|cost|details`, `?palette=1`, `?shortcuts=1`, `?hover=C-WSCR`, `?scroll=dock`, `?focusq=1`,
  `?confirm=requeue`
- Fleet: `?tab=spend`, `?drawer=r-chatgpt`, `?confirm=retire`, `?paused=r-claude`
- Mobile: `?sheet=C-6J9H`, `?tab=cost`, `?tabbar=exceptions` (board) or `?tabbar=spend` (fleet), `?confirm=retire`,
  `?more=1`
- Any desktop mock: `?keys=j,j,Enter` replays keystrokes after load, so a headless render can show keyboard state
