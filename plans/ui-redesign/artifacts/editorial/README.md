# Editorial instrument

## Thesis

The board is the front page of a newspaper of record, typeset by a precision instrument. It has no boxes, badges or
tiles. The type does the work: how heavy and how narrow a line is tells you how much it wants you. The two chunks parked
on a human are the lead stories, set as condensed-black headlines with a deck that says why and what to do. Running work
is a single typeset table, and each row shows its graph as a line of words with the current node heavy. The 53 ready
chunks are set as agate, like the stock tables at the back of a paper: an ambient grey texture you read on demand, where
one keystroke turns it into a full-size list. The page opens with a lede: one sentence that holds the board's whole
state. On the fleet page that sentence is a headline: *Working. Two slots wait on you; OpenAI is pacing.* Colour is kept
for exactly two signals, and paper and ink are first-class in both daylight and night.

## Principles

1. **Hierarchy is typographic.** State maps to weight, width and style, not to a container:

   | State                  | Treatment                                                                        |
   | ---------------------- | -------------------------------------------------------------------------------- |
   | needs human            | serif, condensed, 800 weight, 34–46px headline, red kicker                       |
   | running                | serif, semi-condensed, 500 weight, 15px, one line                                |
   | delivering             | the same, italic: in motion toward master                                        |
   | landed                 | serif, light (300), grey ink: settled                                            |
   | ready, claimable       | sans, extra-condensed, 500 weight, ink: agate you might glance at                |
   | ready, waiting a chain | sans, extra-condensed, light, grey: texture, with `← C-xxxx` naming its blocker  |
   | the node it is on      | heavy, condensed word inside a light run of the graph's node names; `↺n` bounces |

2. **Colour is signal only.** Red (`--human`) means a chunk is waiting on a person. It also marks any ready chunk whose
   blocker is that chunk, so the cost of a parked chunk shows downstream. Ochre (`--pace`) means a subscription is
   pacing itself. Everything else is ink at five strengths on paper. Links are not coloured.
3. **The sentence is the summary.** The KPI strip becomes a lede the hub could write itself ("Two chunks need you, both
   parked at deliver and holding back four ready chunks…"). Numbers sit in prose, with the urgent clause set heavy and
   red. The spend ledger is a single light numeral beside it.
4. **Baseline grid.** Body, meta and table rows sit on an 18px line, agate on 12px and headlines on multiples of 6. A
   2px rule and a serif section head ("Needs you", "Running", "Fleet") replace every panel border. The masthead closes
   with a newspaper's double rule.
5. **Density without boxes.** Whitespace separates the lead stories. Hairlines separate rows. Column rules separate the
   agate. The desktop board shows 2 stories, 8 running, the fleet's occupancy and windows, 10 of 13 landings, the wire,
   and all 53 ready chunks on one 1600×1000 screen, with no scrolling.
6. **Truth over decoration.** Estimated costs keep their `~`. Every age ticks ("waiting 20m 21s", "reported 3s ago",
   "sampled 1m 02s ago"), so you can tell a stale reading from a live one. Zero is stated in words: "No open asks. No
   open gates."

## The pages

- **Board, desktop.** The masthead holds the sections, the project lens, a live clock and the theme, find and keys
  tools. Below it sit the lede and spend ledger. On the left are *Needs you* (a lead story and a second story), then
  *Running* and *Ready* in agate. On the right rail are *Fleet* (an occupancy glyph per environment and three window
  meters), *Landed today* and the *Wire*.
- **Fleet, desktop.** It reads as a status report. The headline is a verdict. Three window gauges share one scale: a bar
  for what is spent, a caret for how much of the window has gone, a big light numeral and a verdict in words. Each
  runner is set as a column story, with its slot table (needs-human rows heavy and red, delivering rows italic), binds,
  version and actions. The rail holds the host's record and environment occupancy. *Since midnight* is a strip chart of
  environments in use per ten minutes, with time parked on a human in red.
- **Board, mobile.** It answers "does the fleet need me?" above the fold. It opens with *Two chunks need you.*, then
  each story with three 44px touch targets (Take over, Requeue, Open PR). Running is a two-line list. Fleet is one
  sentence. Landed and ready are agate paragraphs.
- **Fleet, mobile.** It opens with a three-line verdict, then the gauges with a large numeral and meter, then each
  runner with its slot glyphs and held chunks, then the host as a sentence.

## Interaction model

**Keyboard (first-class).** `j`/`k` (or arrows) walk every chunk on the page in reading order: lead story, second story,
running, ready agate, landed. `↵` opens the **reader**, a sheet on the right with the chunk's facts as prose, the
takeover command and its node history (bounces in italic, the current visit bold). `t` takes over: it copies
`blizzard runner takeover <id>` for the focused parked chunk, or for the lead story if nothing is focused. `o` opens the
PR. `q q` requeues (the second press confirms). `r` reads the ready list: the agate becomes a three-column list at full
size, grouped by epic, with each chain's blocker named. `/` or `⌘K` opens a command line that searches chunks and
commands, ranking needs-you items first. `g b` and `g f` switch pages. `\` switches between daylight and night. `?`
shows the keys. `esc` closes. On fleet, `x` previews the same page at eighty runners. Focus is a 3px ink bar in the
gutter over a faint paper tint, and the bar is red on a parked chunk. There is no glow and no outline box.

**Pointer.** Every row is a target. Hover tints the paper, and clicking opens the reader. Action words in the stories
underline their accelerator letter, as old menus did, so the pointer teaches the keyboard.

**Touch.** Tap targets are 44px. Tapping a story opens the reader as a bottom sheet. Take over copies the command so you
can paste it into a phone terminal. Requeue asks for a second tap. The tab bar is set in type, and Board carries a red
count of parked chunks, so the answer shows from any tab.

**Live.** The clock ticks. Waiting, running and report ages tick each second. Cost accrues on running chunks every few
seconds, and today's ledger moves with it. A digit that moves thickens for a beat (a weight bump, not a colour flash).
New wire lines arrive and fade in on a paper tint.

## Scaling from 8 running to 80

- **Needs you** never shrinks. It is a story list: past two parked chunks, the third and later fall to a numbered run-in
  list under the second story, still in condensed bold. Ten parked chunks is a fleet problem, and the lede says so.
- **Running** goes from one row per chunk to agate grouped by node. Each node becomes a run-in head ("review 14 ·
  pre-push 9 …"), and the chunks under it keep their ticking time and cost. Rows with a bounce count of two or more stay
  at full size, because they are the ones about to need you.
- **Ready** is already built for hundreds. Agate grows by columns, not rows, and `r` is the reading view.
- **Fleet.** Press `x` on the fleet mock to see it working: ten hosts, eighty runners, about five hundred environments
  set in four agate columns. Each runner is one line of slot glyphs. Red blocks and amber "declining" are the only
  things that catch the eye, and a host with a stale report gets an ochre age. The verdict headline and the three gauges
  stay as they are; they summarise, so they do not scale.

## What it drops from the baseline

- The KPI strip. Its numbers live in the lede, and the counts that matter carry weight in the sentence.
- The six kanban columns. Backlog (1) and waiting (0) become words in the ready flag. Done becomes *Landed today*.
- All-caps letter-spaced monospace labels. Labels are sentence-case serif or condensed sans. Mono is kept only for chunk
  ids, the takeover command and version strings, which are literal tokens.
- Card borders, coloured left edges, filled status pills and the HARNESS and ENROLLED badges. These become words in a
  sentence.
- Segmented 40-cell window meters. They become one hairline scale with a spent bar and an elapsed caret, which shows
  pace (spent compared with time gone) instead of only fill.
- The per-event duplicates in the activity feed. The wire shows transitions once, and the events page keeps the full
  log.
- Navy-on-black as the only mode. Daylight is the default; night follows the system setting or `\`.

## Risks and open questions

- **Fonts.** The mocks run offline using locally installed Noto Serif, Noto Sans and Noto Sans Mono in their static
  width and weight instances. They map to `font-stretch` and `font-weight` through `@font-face local()`. Production
  would ship one true variable family with `wdth`, `wght` and `opsz` axes. The mocks fake optical size with tracking. On
  a machine without Noto, the fallbacks lose the width axis, and with it half of the hierarchy.
- **Agate legibility.** 10.5px extra-condensed light type is meant to be texture. Some operators will find it too faint
  on a low-DPI screen. The `r` reading view is the release valve, and the scale may need to come up a step.
- **Prose summaries must be generated.** The lede and fleet verdict read well only if the hub writes them from state, in
  a fixed grammar. A hand-tuned sentence that goes stale would break the board's promise of truth.
- **Two signal colours is very few.** A third signal (for example, a hub that is down) would have to be expressed as a
  headline, not a colour. That is the intent, but it is untested.
- **Invented data.** These items are not in the baseline: C-QWYS's reason and usage, C-6J9H's node history (rebuilt from
  the activity feed), the claim times behind the running clocks, the since-midnight strip and the 80-runner preview. All
  are labelled as invented in the source.

## Files

- `board-desktop.html`, `fleet-desktop.html`, `board-mobile.html`, `fleet-mobile.html`: self-contained mocks with inline
  CSS and JS and no network access. Append `?theme=night` for the night edition.
- `board-desktop.png`, `fleet-desktop.png`, `board-mobile.png`, `fleet-mobile.png`: headless Chrome renders.
- Extra states: `board-desktop-night.png`, `fleet-desktop-night.png`, `board-mobile-night.png`,
  `board-desktop-reader.png` (`↵`), `board-desktop-palette.png` (`/`), `board-desktop-reading.png` (`r`),
  `board-desktop-focus.png` (`j`), `board-desktop-keys.png` (`?`), `fleet-desktop-scaled.png` (`x`), and
  `board-mobile-reader.png` (tapping a story).
