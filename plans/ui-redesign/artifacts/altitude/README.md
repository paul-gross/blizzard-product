# Altitude — one fleet, five fidelities, one line to drive it

## Thesis

Blizzard has one model: chunks moving through a graph on runners on hosts. The baseline splits that model into pages
(board, fleet, chunk detail, events) and makes the operator move between them. Altitude drops the pages. There is one
continuous space, and the operator changes **altitude** within it. Each chunk exists once and renders at whatever
fidelity the altitude calls for: a **pixel** in the field, a **glyph** of four characters, a **row** with its path
through the graph, a **panel** with its full history, and finally a **line** of its live transcript. Board and fleet are
not two screens. They are two **projections** of the same objects (`group status` and `group place`), and `p` flips
between them while each chunk animates to its new home. A query line at the top is the main input. Queries never
navigate: they re-weight the space. Chunks that match keep their fidelity, and the rest drop to pixels in place, so the
operator always knows where they are.

## Principles

1. **Fidelity follows attention, not position.** A chunk that needs a human renders one level above its peers (a full
   card in a row view, a row in a glyph view). Calm work drops a level. Held work collapses into chains. This is the
   direct answer to "53 ready cards bury the 2 that need a human".
2. **Show blast radius, not just state.** Every needs-you card leads with what it holds. C-6J9H holds 24 chunks, and the
   ready region shows them as one chain, "behind C-6J9H". The question "which one first?" answers itself.
3. **One object, one identity.** A chunk keeps its key across every fidelity and projection, so changing zoom or
   projection moves things rather than replacing them.
4. **Colour is signal only.** The page is ink on paper. Vermilion means a human is needed and nothing else. Ultramarine
   means live motion and nothing else. Grey is everything that does not need you. A hatch means held. No colour is
   decorative.
5. **Type carries hierarchy.** Noto Sans's width axis does the work that all caps did in the baseline. Counts are
   extra-condensed and heavy, titles are semi-condensed, and landed work is condensed and light. Mono is kept for ids
   and money. A serif italic (STIX Two) is kept for one thing: sentences an agent or the system wrote to you, such as
   asks and escalation reasons.
6. **Truth over tiles.** Spend, subscription windows and hub health are one line of text with hairline tracks. Each
   track shows spent against elapsed, so being ahead of pace is visible (the bar turns vermilion). There is no KPI
   strip.
7. **Repeats fold.** The baseline feed repeats `C-WSCR pre-push → delivering → deliver` seventeen times. The ticker
   folds repeats (`×17`) and stays on one line.

## The five altitudes

| key | altitude | a chunk is…                                                                  | what you read                                 |
| --- | -------- | ---------------------------------------------------------------------------- | --------------------------------------------- |
| `1` | field    | a pixel (all 990, including 926 historic)                                    | the shape of the whole fleet in one glance    |
| `2` | glyph    | `6J9H` with node and spend, and a cost underline                             | everything live and held, without titles      |
| `3` | row      | id · ref · title · six-cell node track · runner · age · cost                 | the default working altitude                  |
| `4` | panel    | full chunk: path-through-graph diagram, step table, dependencies, issue text | why it is where it is (bounces drawn as arcs) |
| `5` | line     | the step's transcript, streaming                                             | what the agent is doing right now             |

The breadcrumb under the query line is the zoom path (`fleet › xps › r-claude › C-WSCR › step 12 › line 7`). It lights
up to the current altitude. In `4` and `5` the space does not disappear: it shrinks to a glyph-fidelity rail on the
left, and `j`/`k` still walk it. Zooming into a **runner** is itself a query (`runner:r-claude`), so "drill down" and
"filter" are the same operation.

## The query line

```
needs:human | sort holds              what needs you, biggest blast radius first   (@triage)
cost>3 | sort cost                    who is burning                                (@burn)
bounces>2 | sort bounces              who is looping                                (@bouncing)
is:blocked | group epic               held work, by epic                            (@chains)
is:claimable epic:pw                  what PW-HUBTIER can start now
cost>3 | group runner                 an ad-hoc projection
@triage | takeover                    act on every match — a confirm strip names them, ↵ runs
needs:human cost>5 | save @hot        a saved query becomes a view that completes after @
```

Filters are `key:value` and `key>n` (`status needs node runner epic is holds cost age bounces attempts id`), plus bare
words that match against titles and refs. Pipes are `sort`, `group`, `take`, `only` and `save`. A trailing verb
(`takeover requeue answer promote pause open`) turns the query into a command over its matches. Tab completes and ↑/↓
walks the suggestions. Tokens are syntax-highlighted in place, and verbs show in vermilion so a destructive line looks
destructive before you press enter.

## Interaction model

**Keyboard (first-class).** `/` opens the query line. `j k` walk, `J K` jump section, and `g n`, `g l`, `g y`, `g d` go
to needs-you, live, ready and landed. `↵` or `=` zooms in on the focus, and `-` or `esc` zooms out. `1`–`5` set the
altitude and `p` flips board and fleet. `x` selects. `t` takes over, `a` answers, `r` requeues, `u` promotes, `o` opens
the PR and `P` pauses the runner. Each acts on the selection, or on the focus when nothing is selected. `?` lists every
key and `\` switches day and night. In panel view, `j k` walks steps; in line view, it walks transcript lines.

**Pointer (optional).** Click focuses, and a second click or double-click zooms in. Ctrl+wheel (or a trackpad pinch)
zooms around the element under the pointer. The peek pane on the right always shows the focus one fidelity deeper than
the canvas, with its action buttons.

**Touch.** Pinch out or in steps the altitude. A horizontal swipe flips board and fleet, and in panel or line view a
swipe right zooms out. Long-press opens an action sheet. Tap focuses and double-tap zooms. The query line sits at the
bottom, under the thumb, with the altitude ladder beside it. Needs-you cards carry two full-width targets (take over or
answer, then requeue), because on mobile the question is "does the fleet need me?".

## From 8 running to 80

Every region has a fidelity budget. Past about 16 live chunks, the row altitude keeps rows only for **anomalies**
(looping three or more times, spending more than $6, or older than 110 minutes), sorted worst first. Calm work folds to
glyphs. Past three needs-you items, the two with the widest blast radius keep the full card and the rest become compact
rows. Ready work is always chains, so 53 or 530 ready chunks cost the same number of lines: one per root. The fleet
projection scales by runner block, and at glyph or field altitude each runner is a strip of slot pixels.
`board-desktop.html#scale=80` runs this with 80 synthetic live chunks across 10 runners; see `scale-80-live.png`.

## What it deliberately drops

- **The KPI strip.** Counts live in section heads, where they are next to the things they count.
- **Kanban columns.** Six equal columns give equal weight to unequal things. Regions stack by urgency instead.
- **Page tabs** for board, fleet, events and chunk detail. These become altitudes and projections. Events become the
  folded ticker and, at line altitude, the transcript.
- **The open-questions and open-gates panels.** An ask or a gate is a chunk that needs you, so it lives in the needs-you
  region with its question set in the agent's own words.
- **The project lens chips.** Use `project:winter` in the query line, or save it as a view.
- **All-caps letter-spaced mono, pills on every state, and box borders on every panel.**

## Risks

- **Spatial memory against reflow.** When a filter or zoom moves things, the FLIP animation and in-place de-focus help
  keep the operator oriented. But at field altitude, a large live change can still reshuffle the map. The pixel layout
  would need stable slots (by chunk age) to be trustworthy over hours.
- **Query-line literacy.** The grammar is small, but it is a grammar. Saved views and tab completion carry newcomers,
  and every action is still reachable by key and pointer without typing a query.
- **Attention-weighted fidelity can hide things.** A calm chunk folded to a glyph is easy to miss if the anomaly rules
  are wrong. The thresholds must be tunable, and their reasons should be visible ("kept at row fidelity: looping").
- **Light theme as default.** Paper suits long sessions and screenshots, but many operators expect dark. Night (`\`)
  exists and uses a warm black, not navy. Which one is the default needs testing with real operators.
- **Data honesty in the mock.** Titles, ids, costs, runners, the blocking chains and C-WSCR's 12-step history are real
  baseline data. Some data is reconstructed:
  - the other chunks' step histories are shaped from their attempt counts;
  - C-QWYS's question and the transcripts are illustrative;
  - the 80-chunk fleet is synthetic.

## Files

| file                                                          | what                                                                               |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `board-desktop.html` / `.png`                                 | the space in the board projection at row altitude (1600×1000)                      |
| `fleet-desktop.html` / `.png`                                 | the same space in the fleet projection (1600×1000)                                 |
| `board-mobile.html` / `.png`                                  | board projection on a phone (390×844 at 2×)                                        |
| `fleet-mobile.html` / `.png`                                  | fleet projection on a phone                                                        |
| `zoom-1-field.png` `zoom-2-glyph.png`                         | altitudes 1 and 2                                                                  |
| `zoom-4-panel.png` `zoom-5-line.png` `board-mobile-panel.png` | altitudes 4 and 5, and panel on mobile                                             |
| `query-defocus.png`                                           | `is:blocked epic:pw \| sort holds`: matches keep fidelity, the rest drop to pixels |
| `scale-80-live.png`                                           | the fidelity budget at 80 live chunks                                              |
| `board-desktop-night.png`                                     | night theme                                                                        |

Every mock takes hash state for review, for example `#z=4&f=WSCR`, `#q=needs:human`, `#scale=80` or `#theme=night`. The
four HTML files share one generated core and differ only in device and starting projection. They make no network
requests and name system fonts (Noto Sans, Noto Sans Mono and STIX Two Text), with generic fallbacks.
