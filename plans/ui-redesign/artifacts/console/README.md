# Console: classic SaaS, built for operators

**Thesis.** The operator does not need a new metaphor. They need the conventions they already use every day in Linear,
GitHub, Vercel and Datadog: a left nav, a ⌘K palette, saved views over a table, and a detail drawer. Each should be done
precisely enough that blizzard feels like a mature product. Console keeps the baseline's real virtues: density,
everything on one screen, honest cost, and live movement. It replaces the baseline's look with a quiet greyscale shell
in which **hue only ever means status**. Needs-human is the one red thing on the page. It is pinned first, never capped,
and carries its reason and its **Take over** button in the row. The 53 ready chunks sit below it in a group that is
capped and sorted unblocked-first. The design's bet is that a familiar app can feel right in the hand, and that this
matters more than novelty.

## Principles

1. **Colour is signal, chrome is grey.** Buttons, nav, borders and text are neutral. The primary button is ink, not a
   brand hue. Six status roles own the six hues. If something is coloured, it is telling you something.
2. **Attention order, not pipeline order.** Every list, column, palette suggestion and mobile screen orders work the
   same way: needs human, waiting, running, ready, backlog, done. Done is a collapsed group or a saved view. It is never
   a peer of live work.
3. **Shape before colour.** Status glyphs follow Linear's circles. Backlog is a dashed ring, ready an empty ring,
   running a half fill, delivering a three-quarter fill, needs-human a solid disc with `!`, and done a solid disc with a
   check. Each glyph reads in greyscale and has an `aria-label`.
4. **One action away from taking over.** A needs-human row, card, drawer and mobile card all expose **Take over** and
   **Requeue** directly. They are also bound to `T` and `R`.
5. **Honest numbers.** `~` marks estimated cost, exactly as in the baseline, and a tooltip explains it. Counts, cost and
   times use tabular figures. "Holds back 5" counts only blocking edges the hub knows by id. It does not inflate.
6. **Everything the baseline knew, still one keystroke away.** Spend today and yesterday sit in the top bar. The
   subscription windows and the latest event sit in the status bar on every page. Nothing that used to be visible is
   more than one click deep.

## Information architecture

```
[tenant switcher: paul-gross ▾]          ← who/where; tenant settings, members
[Search or jump to…  ⌘K]
Inbox            (2)  red badge           ← needs-human chunks + agent asks + open gates
Chunks                                    ← was "board"; List and Board are layouts of it
Events
── Fleet
Hosts & runners   •   amber dot when a runner is declining
Usage & spend                             ← subscription windows over time, spend by day
── Workflow
Graphs
Gardening
── Projects                                ← the project lens, global and persistent
All projects  64 ✓
■ blizzard    64
■ winter       0
                     (spacer)
Settings & admin                          ← was the "admin" tab
Shortcuts  ?
[PG Paul Gross · Owner]  [⇤ collapse]     ← user menu: profile, compact, theme, shortcuts, sign out
```

- **Inbox gets its own entry, and it alone carries the red badge.** "Does anything need me?" is a different question
  from "show me the work". The asks and gates panels, which sat empty on the baseline board, belong here too. On the
  Chunks page the same set is the **Needs attention** saved view (`2`), so a desk operator never has to leave the table.
- **The project lens lives in the sidebar**, where Linear puts teams. It persists (`localStorage`), scopes every page,
  and appears in the breadcrumb (`paul-gross / All projects ▾ / Chunks`), where it can be switched too. Switching to
  `winter` shows a real empty state.
- **The tenant switcher sits at the top**, the Vercel and Linear position. The **user menu sits at the bottom**.
- **Admin is demoted** to Settings & admin. It is configuration, not daily operation.
- **Collapsed**, the nav becomes a 56 px icon rail. Section labels turn into hairlines, badges and dots stay on the
  icons, and every item shows a tooltip with its label and shortcut on hover or keyboard focus. Toggle it with the
  button, `[`, or `⌘\`. The state persists in `localStorage` (`bz.nav`).

## The Chunks page: a table first, a board when you want one

**List is the default and Board is one key away (`V`).** The baseline's kanban gives every status a column of equal
weight, which is exactly why 53 ready cards bury two that need a human. A grouped table fixes the hierarchy for free:

- The **Needs human** group is pinned first, tinted, and never capped. Each row shows its reason ("Bounce cap crossed
  after 6 bounces · takeover · since 12:48 · holds back 5 chunks") with a dark **Take over** button and a requeue
  button. It is the only place a filled ink button appears in the table.
- **Running** shows a six-segment pipeline stepper (triage → deliver) for where each chunk sits, plus its node, runner,
  PR, cost, and when it entered the node.
- **Ready** is sorted unblocked-first and capped at 16 rows with a "Show 37 more" button. There is a **Hide blocked**
  toggle. Blockers show as `🔒 C-6J9H needs human`, and the id turns red when the blocker is the halted chunk.
- **Done** starts collapsed.
- **Saved views** are tabs with counts: All open, Needs attention, In flight, Ready queue, Done · 48h (`1`–`5`), plus "+
  View". Filter chips (Status, Node, Runner, Epic, Cost), Group and Order complete the classic toolbar. `/` focuses the
  text filter.
- **Five stat cards** replace the eleven-counter KPI strip. Each one is clickable and applies the matching view. The
  needs-you card is the only red one.

The **Board** layout is kept because the flow-at-a-glance reading is genuinely useful on a wall monitor. Its column
order is the attention order, and the needs-human column is tinted. Ready splits into *Unblocked* and *Blocked*
sub-lists, and the empty Waiting column collapses to a 44 px spine.

**The detail drawer** (520 px) pushes the table rather than covering it. Columns the narrower table cannot hold are
hidden: runner, PR, since and the inline ref. The pipeline stepper, the Take over label and the toolbar's Epic, Cost,
Group and Order labels also fold away, so titles keep the width. The drawer has a status badge, id, previous and next
(`K` / `J` move through chunks while it stays open), "open as page", and close (`Esc`). Its actions are Take over,
Requeue, PR, and a ⋯ menu that holds *Cancel chunk…*, which asks you to type the id. There are four tabs:

- **Overview** carries the needs-human callout with the hub's verbatim reason, a properties list, dependencies as
  clickable chips, a five-cell usage strip, and the work items.
- **Node history** lists every visit with its verdict, tokens, cost and time. Bounces are tinted and halts are red.
- **Artifacts** and **Transcripts** are stubs.

## Fleet

One overview page answers "can the fleet work right now?". The Hosts, Runners, Workspaces, Enrollment tokens and Audit
log tabs are each the full table of one object type.

- **Subscription windows.** One meter per window (Anthropic 5h, Anthropic 7d, OpenAI 7d). The fill shows spend and an
  ink tick shows how much of the window has elapsed, so **fill past the tick means over pace**. Each meter has a
  sentence beside it: "Pace 0.17× — under budget", or "⚠ r-chatgpt declining until 14:41".
- **Host card (xps).** Last report, which counts up live, plus URL, token state, agents and registration. Its actions
  are Enroll, Revoke token (confirm), and **Retire host**, which is destructive and requires typing `xps`.
- **Runners table.** State badge (Working / Declining until 14:41 / Paused), harness, subscription, a per-slot
  environment bar coloured by chunk status, a "2 need you" badge, binds, and last seen (live). Pause and Resume work
  inline (`P`). The ⋯ menu holds edit bindings, revoke token (confirm) and retire runner (type the name). Selecting a
  runner opens the drawer with its throttle explanation, binds and held chunks, each of which links to the chunk.
- **Workspaces and environment slots.** One row per workspace and one tile per slot. A tile shows the chunk, its node
  and its runner. Needs-human tiles are red, and free slots are dashed. Clicking a tile opens that chunk's drawer on the
  Chunks page.

## Compact mode

Compact mode is a single attribute, `html[data-density]`, that every component reads through tokens. Toggle it with
`⇧D`, the top-bar button, the user menu, or ⌘K. It persists as `bz.density`.

| Token        | Comfortable | Compact |
| ------------ | ----------- | ------- |
| Table row    | 40 px       | 28 px   |
| Body text    | 13 px       | 12 px   |
| Meta text    | 12 px       | 11 px   |
| Controls     | 32 px       | 26 px   |
| Page gutter  | 24 px       | 16 px   |
| Card padding | 16 px       | 10 px   |
| Top bar      | 52 px       | 42 px   |
| Nav item     | 32 px       | 26 px   |
| Group header | 36 px       | 28 px   |

Compact also folds a few things away:

- Stat cards become one-line chips.
- The needs-human reason line folds into the row. The row stays red-barred, and the reason is in the drawer.
- Kanban titles clamp to one line.
- Slot tiles go single-line.
- On mobile, the header, chips, needs-you cards and runner cards tighten. About five running rows show below the two
  needs-you cards instead of two, and touch targets stay at least 32 px. See `board-mobile-compact.png`.

At 1600×1000 the visible chunk rows go from 12 to 23, and the needs-human rows keep their red bar. That is the density
the baseline had, now with hierarchy. See `board-desktop-compact.png`.

## Design tokens

**Type.** Inter (variable, inlined as woff2, OFL) for everything. A monospace stack (ui-monospace, SF Mono, JetBrains
Mono, Roboto Mono, Menlo, Consolas) is used only for ids, refs, URLs and versions. Numbers use tabular figures.

| Token | Size | Use                                  |
| ----- | ---- | ------------------------------------ |
| `2xs` | 10.5 | epic tags                            |
| `xs`  | 11   | captions, status bar, column headers |
| `sm`  | 12   | meta (comfortable), body (compact)   |
| `md`  | 13   | body (comfortable)                   |
| `lg`  | 14   | drawer title (compact)               |
| `xl`  | 16   | drawer title, modal title            |
| `2xl` | 20   | page title                           |
| `3xl` | 24   | stat values                          |

Weights are 400 (body), 500 (labels, titles in rows) and 600 (headings, counts). No all caps anywhere.

**Spacing.** A 4 px base: `2 · 4 · 8 · 12 · 16 · 20 · 24 · 32`. **Radii.** 4 (tags), 6 (controls), 8 (cards), 12 (modals
and palette), and full (badges).

**Colour roles.** Light is the default; dark uses the same role names.

| Role      | Glyph              | fg / bg (light)       | Means                                       |
| --------- | ------------------ | --------------------- | ------------------------------------------- |
| `nh`      | solid disc + `!`   | `#c22a31` / `#fdeeee` | A human is needed. Only red thing on screen |
| `wait`    | ring + clock hand  | `#94570a` / `#fdf3dc` | Waiting, throttled/declining, bounce        |
| `run`     | half-filled ring   | `#0b5fc4` / `#ebf3fe` | Running at a node                           |
| `dlv`     | ¾-filled ring      | `#0b7469` / `#e4f5f2` | Delivering (landing a PR)                   |
| `done`    | solid disc + check | `#1b7443` / `#e8f5ed` | Done; healthy; under budget                 |
| `ready`   | empty ring         | `#464c56` / `#eef0f3` | Ready / queued (deliberately grey)          |
| `backlog` | dashed ring        | `#a8afba`             | Backlog                                     |
| `focus`   | 2 px ring          | `#2563eb`             | Keyboard focus. Never used for status       |

Neutrals:

- Canvas `#f6f7f9`, surface `#fff`, subtle `#f2f4f6`, borders `#e3e6ea` and `#cdd2d9`.
- Text `#16181d`, `#464c56` and `#646b76`.
- Ink buttons `#16181d`.

Project marks are iris (blizzard) and plum (winter). They are deliberately off the status hues.

**Contrast (WCAG).**

| Pair                                  | Ratio     |
| ------------------------------------- | --------- |
| Tertiary text on surface (light)      | 5.4:1     |
| Tertiary text on canvas (light)       | 5.0:1     |
| Every status fg on its own bg (light) | 5.0–5.4:1 |
| Every status fg on its own bg (dark)  | 7.1–8.9:1 |
| Focus ring (light)                    | 5.2:1     |
| Focus ring (dark)                     | 6.5:1     |

Status is never carried by colour alone: glyph shape, label, and position (pinned first) carry it too.

**Theme default: light**, following the OS if it asks for dark. `⇧T` toggles it, and the choice persists. The brief
calls the baseline "a 2024 dark dashboard". A cool neutral light theme gives small dense text its best legibility, and
its tinted status backgrounds read clearly. It is also the reference look of the products this direction draws on. It is
also cool, not the warm paper the other proposals converged on. Dark is fully specified for the second-monitor and night
use case. See `board-desktop-dark-history.png` and `fleet-desktop-dark-compact.png`.

## Interaction model

**Keyboard.**

- Anywhere:
  - `⌘K` / `Ctrl+K`: palette.
  - `[` or `⌘\`: sidebar.
  - `⇧D`: compact mode.
  - `⇧T`: theme.
  - `?`: shortcut sheet.
  - `G` then `I` / `C` / `F`: go to Inbox / Chunks / Fleet.
- Lists:
  - `J` / `K` and the arrow keys: move the row cursor. The cursor appears on first use and has a visible inset ring.
  - `↵` or `O`: open the drawer.
  - `Esc`: close the drawer.
  - `/`: filter.
  - `1`–`5`: switch saved views.
  - `V`: List or Board.
- On a chunk: `T` takes over, `R` requeues (confirm), `P` opens the PR, `C` copies the id.
- Fleet: `J` / `K`, `↵` opens a runner, `P` pauses it.

Menus support arrow keys, and `Esc` always unwinds the topmost layer first.

**The palette.** It opens with the two needs-human actions as *Suggested* ("Take over C-6J9H — holds back 5"). It
searches every chunk by id, ref, title or epic: `6j9h` or `#1449` works. It also runs navigation, preference,
project-lens, view and fleet commands, with their shortcuts shown.

**Pointer.**

- Hover reveals row affordances.
- Tooltips show the shortcut for every icon button and every collapsed rail item.
- Clicking a stat card applies its view.
- Clicking a blocker chip in the drawer jumps to that chunk.
- Clicking a slot tile on Fleet opens the chunk on Chunks.

**Touch (mobile).**

- **Bottom tab bar:** Inbox (red badge), Chunks, Fleet (amber dot), Events, More. A tab bar keeps the badge permanently
  visible and in thumb reach. A drawer would hide the only badge that matters behind a hamburger. Five slots fit what an
  operator away from the desk reaches for, and Usage, Graphs, Gardening, Settings, compact mode and theme go under More.
- **Chunks:** a large title with a one-line summary ("2 need you · 8 running · 10 / 14 slots · ~$114.44 today"), filter
  chips with counts, needs-you **cards** with full-width Take over and Requeue buttons, running rows, ready folded to
  three unblocked chunks plus "Show all 53", and done behind a link.
- **Detail** opens as a bottom sheet with a sticky action footer.
- **Fleet:** windows, runners (Pause / Details), slots, and the host.
- **Destructive actions** use an iOS-style action sheet. Retire requires typing the host name.
- The tenant and project lens are a sheet off the app bar.

**Live.** The status bar clock ticks, and the "Last event" slot replays real events every few seconds. Rows flash once
when their chunk reports, then stay still. Host and runner heartbeats count up. Nothing pulses or spins.

## Scaling from 8 running to 80

- **Needs human is never capped.** At 15 it is still first, and the Inbox badge says 15. The summary card shows the
  oldest halt and the total held back.
- **Running at 80** is grouped by runner or node (`Group:` menu), sub-groups collapse, and rows are virtualized. The
  pipeline stepper makes a node-sorted column of 80 scannable. The running card's slot bar becomes proportional ("62 /
  80").
- **Ready at 500** stays capped and unblocked-first. Blocked chunks hide behind one toggle. Search, saved views and
  filter chips do the rest.
- **Fleet at 10 hosts and 30 runners.** The host card becomes the Hosts table, with the same columns as Runners. The
  runners table sorts by attention (needs you, declining, paused, then working). Slot rows stay one per workspace, and
  tiles wrap. In compact mode they become single-line chips, so about 60 slots fit on one screen.
- **The palette and URL-addressable state** (`?view=`, `?drawer=`) make any of these one keystroke or one link away.

## What it deliberately drops from the baseline

- **The eleven-counter KPI strip.** It becomes five clickable stat cards. Spend sits in the top bar, and hub health is
  the "Live · hub ok" pill.
- **The right rail** of runner meters, open questions, open gates and the activity feed. Meters move to the status bar
  and Fleet. Asks and gates move to Inbox. The feed becomes Events plus the status bar's last event and row flashes.
- **Done as a sibling column of live work.** It becomes a collapsed group and a saved view.
- **One monospace face at one weight in all caps.** Inter at three weights takes over, with mono kept for ids.
- **Decorative colour.** Teal titles, orange costs and cyan links go. Links are ink, and costs are plain tabular
  numbers.
- **Flat top tabs.** They become a grouped left nav with Admin demoted.

## Risks and what I am least sure of

- **Familiar may read as generic.** The craft lives in the details, and a reviewer skimming a PNG may see "another
  Linear clone". The proposal is only as good as its execution in the real Angular board.
- **The table loses the kanban's gestalt.** The kanban stays one key away, but the flow reading is secondary now.
- **The "holds back" definition.** "5" counts only edges known by id. Other proposals said 4, 23 or 24. A build has to
  pick one definition (direct dependents or full transitive closure) and label it.
- **Light by default** may disappoint an owner who lives in dark terminals. Following the OS softens this.
- **Inbox and Needs attention overlap.** Two routes to the same two chunks is deliberate redundancy, but it could
  confuse.
- **Mock data.**
  - C-QWYS's escalation reason is inferred, because the captures show only its status.
  - The window percentages were read off the baseline meters.
  - Saved-view authoring, filters, Display, and every nav section other than Chunks and Fleet are stubs that explain
    themselves with a toast.

## Files

| File                               | What                                                           |
| ---------------------------------- | -------------------------------------------------------------- |
| `board-desktop.html` / `.png`      | Chunks, list layout, comfortable, light (1600×1000)            |
| `fleet-desktop.html` / `.png`      | Hosts & runners overview (1600×1000)                           |
| `board-mobile.html` / `.png`       | Chunks on mobile (390×844 @2x)                                 |
| `fleet-mobile.html` / `.png`       | Fleet on mobile (390×844 @2x)                                  |
| `board-desktop-compact.png`        | Compact mode, nav expanded                                     |
| `board-desktop-nav-collapsed.png`  | Icon rail with tooltip on Inbox, keyboard row cursor on C-27A6 |
| `board-desktop-drawer.png`         | Drawer open on C-6J9H (needs human)                            |
| `board-desktop-palette.png`        | ⌘K palette with suggested takeovers                            |
| `board-desktop-kanban.png`         | Board layout                                                   |
| `board-desktop-dark-history.png`   | Dark theme, drawer on C-WSCR's node history with bounces       |
| `fleet-desktop-drawer.png`         | Runner drawer on r-chatgpt (declining)                         |
| `fleet-desktop-retire-confirm.png` | Type-to-confirm on Retire host                                 |
| `fleet-desktop-dark-compact.png`   | Fleet in dark, compact, rail                                   |
| `board-mobile-sheet.png`           | Detail sheet for C-6J9H                                        |
| `board-mobile-compact.png`         | Mobile in compact mode                                         |
| `board-mobile-inbox.png`           | The Inbox tab                                                  |
| `fleet-mobile-retire-confirm.png`  | Retire action sheet                                            |

Every mock is a single offline file and accepts URL state for review:

- `?nav=collapsed`
- `?density=compact`
- `?theme=dark`
- `?view=attention|flight|ready|done`
- `?layout=board`
- `?drawer=C-6J9H` (or `r-chatgpt` on Fleet)
- `?tab=history`
- `?palette=1`
- `?sheet=C-6J9H` (mobile)
- `?confirm=retire`
