# Synop: the fleet as a weather chart

## Thesis

Forecasters read a synoptic chart (hundreds of station plots, fronts and isobars) faster than anyone reads a table.
Every station is drawn as the same compact glyph, with each variable in a fixed position, so the eye learns *where* to
look rather than *what* to read. A fleet of agents behaves like weather. Many stations report at once. Systems form,
move, and stall. Some things build up and then discharge. Pressure, in this case money, accumulates. Synop plots every
chunk as a **station** on a fixed map of the backlog. Each epic is a piece of land, and chunks with no epic sit at sea.
What is happening *now* is drawn as weather over those stations: rain where agents are working, thunder where a human is
wanted, and fronts where one chunk holds another. The fleet itself is the **upper-air chart**: subscriptions are air
masses with soundings, runners are stations aloft, and workspaces are lakes. On the phone the same chart is **read
aloud** as a shipping forecast. The notation is learned once, from one page (`legend.html`). After that, an operator
reads the state of the fleet in about one glance.

## Principles

1. **Fixed positions, not labels.** A station plot carries up to eight variables with no field names: cost (upper left,
   italic when estimated), call sign (upper right), state (left), barograph (right), minutes in the current node (lower
   left), bounces (lower right), and the work item below. A barb's bearing names the runner, and its feathers count
   dollars. Once learned, a plot reads faster than a table row because nothing on it is a label.
2. **A map you memorise.** Places do not move while you are away. Settled lands drift to the west margin and work still
   to come lies east. Inside a land, blockers sit west of what they hold. After a week the operator knows that Strict is
   mid-left and the Hub-tier fan is bottom-centre. Spatial memory replaces scrolling and filtering.
3. **Weather is the signal; ground is the context.** The paper, land and settled dots are quiet. Colour is reserved for
   meaning, and every colour has exactly one job. Red is thunder (a human is wanted) or a warm front (its blocker is
   running). Blue is a cold front (its blocker is only ready). Green stipple is precipitation radar, sized by cost.
   Graphite is the operator's own pencil. Needs-human no longer weighs the same as done: a red ring and hatched cell on
   a quiet chart is the loudest thing on the screen.
4. **Fronts carry the dependency graph without drawing a graph.** A front's kind says whether its chain will move.
   **Warm** (red semicircles) means the blocker is running, so it will move soon. **Cold** (blue triangles) means the
   blocker is only ready, so nothing moves until someone claims it. **Stationary** (alternating) means the blocker is
   stuck on a human and the whole chain behind it is frozen. In the real data, one stationary front at 6J9H freezes 24
   chunks across Forges, Flows, Policies, Strict, Hub shoals and one ship at sea, which the baseline showed only as a
   chain of "BLOCKED BY" strings.
5. **The barograph shows the walk.** The tendency trace is the chunk's node level over its last eight visits. A clean
   climb, one recovered bounce, a saw-tooth ping-pong (6J9H: deliver and pre-push, six bounces) and a cliff (WSCR thrown
   from pre-push back to build) are four shapes that are recognisable at 20 px.
6. **Writing is part of the notation.** Every operator action is written in **pencil** beside the station, with initials
   and time ("pg 13:08 takeover"). It turns to **ink** when the hub confirms it. A front you draw is a dashed graphite
   front until the hub accepts the dependency. What you asked for and what has happened are never confused.
7. **The chart teaches itself.** The plot slip in the margin redraws the selected station at twice the size with each
   position labelled in faint italic, and decodes it in one sentence. The corner key holds runner bearings, the barb
   scale and the three front kinds.

## The four surfaces

| File                 | What it is                                                                                                                                                                                                                                                                                                  |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `board-desktop.html` | **Surface analysis.** 102 stations from the real board: 2 thunder, 8 precipitating (2 landing), 53 fair, 1 backlog square, and 38 settled dots in the western islands. 41 fronts (36 cold, 4 warm, 1 stationary). Spend isopleths. The margin holds the warnings, the plot slip, and a teletype (live feed) |
| `fleet-desktop.html` | **Upper air.** Anthropic and OpenAI as air masses, each with a sounding of its windows. Host xps as land. Workspaces as lakes, with each slot holding the surface plot of its chunk. Runners as stations aloft. Rivers are binds (dashed dry beds when 0 are working), and jets are burn                    |
| `board-mobile.html`  | **The forecast.** Gale warnings (storms that want you, with take-over/requeue as 44 px buttons), a general synopsis composed from the chart, then each land in fixed order with a one-line forecast and a strip of tiny plots. Tapping a land opens its own chart, which pans sideways                      |
| `fleet-mobile.html`  | **Coastal reports.** Each subscription as a one-line horizontal sounding (elapsed vs spent, spread or *saturated*). Each runner as a station report with its plot, a spoken report, its held chunks and a pause button. Lakes as slot strips. Host report with its three actions                            |
| `legend.html`        | The one-page primer. It is drawn by the same code as the mocks, so it cannot drift from them. `?` opens it inside every surface                                                                                                                                                                             |

Additional renders: `board-desktop-writing.png` (QWYS taken over and inked; a pencil front being drawn from HBRY),
`board-mobile-areas.png` (sections folded, Forges and Strict charts open), `board-mobile-sheet.png` (station sheet with
actions) and `fleet-mobile-lower.png` (lakes and host).

## The notation in brief

- **Circle**: eighths of the graph walked, drawn like cloud cover (ready 0 to done 8). A red ring means it wants a
  human. A square means backlog.
- **State glyph**: rain = running, shower = landing, thunder = needs human, fog = waiting or saturated, ‖ = held by the
  operator, nothing = fair.
- **Barb**: the bearing is the runner (r-claude NNW, r-chatgpt NNE). A half feather is $1, a full feather $2 and a
  pennant $10. A dot means under $1.
- **Barograph**: the node level over the last eight visits. Red dots mark bounces, and the lower-right number counts
  them.
- **Fronts**: warm, cold, stationary or pencil, starting at a dot on the blocker. Pips sit on the left as you walk the
  front from blocker to held.
- **Upper air**: T (red) is the % of the tightest window elapsed and Td (green) is the % spent, at the positions where a
  station model puts temperature and dew point. **When Td meets T the air is saturated**: fog, and the runner declines
  work. The spread T−Td is the headroom. In the real data, OpenAI 7d is at 7/8, saturated (r-chatgpt is declining until
  14:41), and Anthropic 7d is at 14/12, a spread of 2 and near saturation, though its 5h window is wide open.

## Interaction model

**Keyboard (desktop, primary).** Navigation is spatial, because the map is the index.

| Keys                | Does                                                                                     |
| ------------------- | ---------------------------------------------------------------------------------------- |
| `g` + 2 chars       | go to a call sign (like an ICAO lookup); two characters are almost always unique         |
| `h j k l` / arrows  | walk to the nearest station in that direction                                            |
| `n` `N` · `.` `,`   | next/previous storm · next/previous precipitation                                        |
| `[` `]`             | walk a front up to its blocker / down to what it holds                                   |
| `1`–`9`             | jump to a land by its number                                                             |
| `t` `q` `p` `P` `o` | take over · requeue · hold/release · promote backlog · open PR (pencil, then ink)        |
| `d` … `⏎`           | draw a front (declare a dependency) from here to the station you walk to                 |
| `u`                 | rub out the last unconfirmed pencil                                                      |
| `=` `-` `0`         | zoom around the selection · out · whole chart                                            |
| `F` / `S`           | upper-air (fleet) chart / surface chart; `tab` walks stations aloft; `p` pauses a runner |
| `?`                 | the legend                                                                               |

**Pointer.** Hovering a station decodes it in the slip. Clicking selects it. Shift-dragging from one station to another
draws a front. The wheel zooms around the cursor. On the fleet chart, double-clicking a slot opens that chunk on the
surface chart.

**Touch (phone).** The forecast is read top to bottom: if the gale warnings say *none*, the operator is done. Section
headings fold. Tapping a land opens its chart, cut from the surface analysis at the same positions, and it pans sideways
under the thumb. Tapping a station raises a sheet with its decode and large take-over / requeue / hold / PR buttons.
Writes show in pencil until the hub confirms.

**Live.** The valid time, minutes-in-node and report ages tick every second. Burn accrues. A scripted series of reports
arrives: MFJE moves build → review, FC1D moves build → review, BWA2 moves mutation → pre-push, then **PTP2 lands**, and
its warm front to JME3 disappears, so JME3 becomes claimable. Each report flashes a ring on its station and prints on
the teletype.

## Scaling from 8 running to 80

The chart scales the way weather charts do: by area, not by length. At 8 running, individual plots are read. At 80,
precipitation shading merges into **systems**, and the operator reads the systems ("a wet band across Hub-tier and
Strict, dry elsewhere") before zooming in (`=`) to read plots. Fronts likewise group into fans (the P79Q fan of 12 is
already one shape, not 12 lines), and stationary fronts matter more as the count grows. The phone forecast stays the
same length, because it grows by land, not by chunk: each land gets one line and a strip. Hosts become an archipelago on
the upper-air chart, with lakes per workspace. Settled stations older than 48 h leave the chart, so the 926 done never
accumulate. The lands they leave behind sink to one line: "Playwright, Actions, Notes, Flags, Lines, Release: settled."

## What it deliberately drops

- **The KPI strip.** Counts become one sentence (the census) and the synopsis. Tiles that weigh 926 done the same as 2
  needs-human are gone.
- **Columns by status.** Status is the weather on a station, not the station's address. A station keeps its place
  whatever its status.
- **Titles on the board.** Titles live in the slip, on hover or selection. The chart shows the call sign and work item
  only. This is the largest density trade.
- **The activity feed as the primary live signal.** Live change is drawn on the map (flash rings, fronts vanishing). The
  teletype is the record, not the instrument.
- **Tabs for graphs, events, gardening and admin**, and the per-chunk transcript and artifact tabs. They are out of
  scope here. The surface and upper-air charts are the two "levels" of the same atmosphere, switched with `S` and `F`.
- **Monospace, all caps, dark navy.** The chart is printed paper: a narrow grotesque for plotted figures, a schoolbook
  serif for lettering, and italics for water and for estimates.

## Risks

- **Learning cost is real.** The notation pays back only for daily users, and nobody can use it before reading the
  legend. Mitigations: the self-labelling slip, the corner key, and a phone surface that is plain sentences.
- **Layout stability against churn.** The promise that places do not move is easy with 14 lands. It gets harder as epics
  are minted and closed hourly. The layout must be incremental (new lands appear east, closed lands sink west) and never
  globally re-flowed, or the operator's spatial memory is broken.
- **Fronts can saturate.** A chunk blocked by six draws one front, and the others live in the slip. A dense cross-epic
  dependency web could still turn the sea into a net. A fold rule (one front per land pair, with a count) is the
  fallback.
- **The metaphor can mislead in edge cases.** Barb bearings encode runner identity, which works for 2 to 6 runners but
  not 40. Past that, bearing should encode the *subscription* and the runner should move to the slip.
- **Isopleths are the weakest element.** They are true (dollars of today's spend within reach) but rarely acted on. They
  should be the first thing cut if the chart feels busy.
- **Accessibility.** The chart is visual. The phone forecast is the accessible rendering of the same state, and the
  slip's one-sentence decode exists for screen readers, but the desktop chart itself needs a parallel text model.
