# Desk: the board as a shared inbox

**Thesis.** The operator's real job on the board is triage. A chunk halts, an agent asks, a gate opens, and each one is a
conversation the fleet has started with a human. Support teams already have a mature tool for this exact loop: Front,
Intercom, Zendesk, Help Scout and Superhuman. Desk borrows that product family rather than the issue tracker. It has
three panes: views on the left, a queue of conversations in the middle, and the open conversation on the right with a
context sidebar. The chunk's node history becomes the conversation thread. Every item that waits on a human carries an
SLA clock, can be assigned or snoozed, and is resolved with a macro (take over, requeue, rebase and retry, raise the
bounce cap). The single-key triage comes from Superhuman: `E` resolves, `H` snoozes, `Z` undoes. The goal state is inbox
zero, and inbox zero is also where the operator walks away. So that screen answers the next question: what will the fleet
do without me, and when will it need me again? Running, ready and landed work is still one click away, but it is the
fleet's work, not the operator's. Desk is a sibling of Console. It uses the same cool neutral chrome and the same rule
that hue only means status, with needs-human as the one red thing. It is organised around a different question: not
"what is open, and in what order?" but "what is waiting on me, and how fast can I clear it?"

## Principles

1. **What needs a human is a queue, and the queue drains to zero.** Needs-human chunks, agent asks and open gates share
   one inbox, oldest first. Resolving an item removes it, and the next item opens by itself. An empty queue is a
   finished task, and the screen says so.
2. **The thread is the history.** A chunk's visits, bounces, PR events and the hub's escalation are shown as messages in
   time order. A run of identical bounces collapses into one stack, the way a support tool folds repeated messages. The
   escalation that needs you is the newest message, set with a red edge, and it carries the hub's reason word for word.
3. **The clock is visible.** Every waiting item shows how long it has waited, with a ring that fills toward a response
   target (30 minutes by default). Taking an item, by assigning it or taking it over, counts as the first response and
   stops the clock. Snoozing hides an item but does not stop its clock. Desk says this every time you snooze.
4. **Resolutions are macros, and your note travels with them.** A macro is a saved resolution: take over, requeue, rebase
   and retry the land, raise the bounce cap, send back to build, cancel. The composer under the thread holds a note for
   the next agent, and whichever macro you send carries it. Each conversation has a suggested macro, taken from the hub's
   own advice, and `E` sends it.
5. **Hue only means status, and chrome stays grey.** Needs-human is the only red on the page. Amber means pacing or a
   bounce, blue means running, teal means delivering, green means landed. The single filled ink button on a
   conversation is its suggested resolution.
6. **Prose before counts.** A computed sentence replaces the KPI strip: "Two chunks need you and hold back 5 others; the
   oldest has waited 21m. The other 8 are running on 10 of 14 slots, r-chatgpt is pacing itself until 14:41, and today
   has cost ~$114.44." Each clause links to the view it summarises.
7. **Always say what happens if you leave.** A forecast of the walk-away sits one key away (`W`), in the queue's footer,
   on the inbox-zero screen and on the mobile home screen.

## Information architecture

```
[paul-gross · blizzard hub ▾]
[Search or run…  ⌘K]
Inbox
  Needs you          (2)  red count     ← needs-human chunks, agent asks, open gates; oldest first
  Mine                0                 ← assigned to you or taken over, until you hand it back
  Snoozed             0                 ← returns to Needs you when the snooze ends
  Handled today       0                 ← what you resolved, with the macro you used
Work
  In flight           8                 ← running and delivering, latest activity first
  Ready queue        53                 ← claim order: unblocked first, then blocked
  Backlog             1
  Landed · 48h       38
Team                         10 / 14 slots
  Fleet          2 online
  RC r-claude   ▮▮▮▮▮▮                  ← presence: status dot, one bar per slot, red where a slot waits on you
  RG r-chatgpt · until 14:41  ▮▮▮▮
  Usage & spend   ~$114
Projects
  All projects 64 · blizzard 64 · winter 0
                (spacer)
Activity · Settings & admin
[PG Paul Gross · On desk ▾]             ← availability: On desk / Away (notify me)
```

- **Inbox and Work are deliberately separate.** Inbox holds what waits on you. Work holds what the fleet is doing. In
  support tools these would be "your inboxes" and "team views". The red count appears only on Needs you.
- **The runners are listed as teammates.** Support tools show who is online and how loaded each person is. Desk does the
  same for runners: a status dot (green working, amber pacing, hollow paused), a bar per environment slot, and a red bar
  where a slot is parked on a chunk that needs you. The fleet is always visible, so there is no status bar.
- **Availability is a real control.** "On desk" and "Away" come from support tools, where an agent's status decides who
  gets routed work. Here, going Away tells blizzard to notify you, and the forecast shows what will happen while you are
  gone.

## The board: Needs you

**The top bar** holds the computed sentence, which may wrap to two lines. To its right are a Live pill (hub ok,
yesterday's spend), the **Walk away?** forecast button, density and theme.

**The conversation list** (middle pane) has a title and count, an order menu (Oldest first, Most held back, Newest), and
filter chips: All, Halts, Agent asks, Open gates. Each item shows:

- the status glyph, the id, the epic tag and the node
- the SLA ring and how long the item has waited, which turns into a filled red pill once it passes the target
- the title, bolded with an ink dot until you have opened it, as an unread conversation is
- the hub's short reason in red ("Bounce cap crossed after 6 bounces · takeover")
- what it holds back ("holds back 5"), the runner, the cost, and the assignee ("Unassigned", or your avatar)

Below a short queue, a quiet **Not waiting on you** block carries the rest of the work in three rows: 8 in flight with a
bar per chunk, 53 ready (12 unblocked, 5 held by C-6J9H), and 18 landed today with spend. The footer always shows the
walk-away line: "Walk away now: next likely needed ~13:45 for C-WSCR".

**The open conversation** (right of the list):

- **Header.** The status badge, id, ref and epic, with PR, ⋯, "1 of 2" and previous/next. Below the title sit the
  actions: Take over `T`, Requeue `R` (or Rebase & retry, when that is the suggestion), Snooze `H`, Macros `;` and
  Assign `A`.
- **Thread.** It opens with the item the chunk was minted from, as an inbound message, with the issue text one click
  away. Folded earlier visits come next ("18 earlier visits · triage → build → review → mutation, 2 bounces · $6.71"),
  then one line per visit: the node agent's avatar, verdict, target node, tokens, cost and time. PR events follow, then
  the bounce stack ("Bounced 5 times between deliver and resolve · visits 22–26 · $0.96", which expands), a divider
  ("12:48 · waiting on a human"), and the **blizzard hub** escalation with its reason, the chunks it holds back as
  chips, and the suggested resolution. Anything you do is added to the thread as a message from **You**.
- **Composer.** "Write guidance for the next agent — it travels with your macro. Type / for macros." Its primary split
  button is the suggested macro (`E`), with the other macros on the caret.

**The context sidebar** (right) shows:

- **Waiting on you.** A large live timer ("21m since 12:48"), a bar toward the target, "Holding back 5 chunks (2
  directly)", and the assignee with **Assign to me** `I`.
- **Details.** Status, node with the pipeline stepper, runner, attempts, graph and project.
- **Pull request.**
- **Dependencies.** Depends on, blocks, and how many more are held downstream through the chain.
- **Cost.** A six-cell usage grid.
- **Runner.** r-claude's card, with its own subscription meters.
- **Work items.**

**Triage, end to end.** Press `E` on C-6J9H and the take-over sheet opens with `blizzard runner takeover C-6J9H` and a
copy button, plus the hub's fallback advice. Confirm, and the item slides out to **Mine**, the clock stops ("SLA met in
21m"), and C-QWYS opens. Press `E` again and C-QWYS is rebased and retried. The queue is now empty, and the reading pane
becomes the full forecast. Every step leaves an Undo toast (`Z`).

**Mine.** A taken-over chunk stays here, marked "You hold it · session open in your terminal", until you hand it back
(`E` on a Mine item is **Hand back to the fleet**).

**Snooze** (`H`) offers 15 minutes, 1 hour, "Until r-claude frees a slot (~13:24)", until 15:00, and tomorrow at 09:00.
Each option has a number key. The footer warns: "Snoozing hides it, it doesn't pause it. The SLA clock keeps running and 5
chunks stay held back."

**Inbox zero** shows "You're clear" in the list, with what you handled and the macro you used. The reading pane shows the
full forecast (see the next section), and the context pane changes to **Fleet right now**: runners, windows, and today's
landed count and spend.

### If you walk away now

Borrowed from Tacit and recast as a plain support-tool panel. It is rewritten every minute from node durations and pace
windows:

- **A headline.** "The fleet keeps working, but C-6J9H and C-QWYS stay parked, and C-6J9H holds 5 chunks of the FORGES
  chain behind it." Once both are cleared, it becomes "The fleet works on its own until about 13:45."
- **A paragraph.** How much lands and what it costs, who is idle and why, and when the fleet next needs you.
- **A short timeline.** "~13:13 C-PTP2 lands PR #1492 and frees C-JME3" through "14:41 r-chatgpt resumes claims". The
  one predicted moment that may need a human is tinted.
- **"Notify me while I'm away when…".** Toggles for: a chunk needs a human, an item waits past 30 min, spend passes $150,
  a runner goes offline.
- **Go away** (`⇧A`). This sets your availability to Away.

Every forecast time carries `~`, and the panel says these are estimates, not promises.

### The rest of the work

The Work views use the same list-and-conversation shell, so the operator learns one layout.

- **In flight.** Each item shows the stepper and node, a live preview line from the runner ("mutation · 41 of 58 mutants
  killed") that updates and flashes once as reports arrive, bounces, PR and cost. The open conversation shows the full
  node history. At its foot is a typing indicator ("r-claude is working at review · 2m 03s") and a status line: "The
  fleet has this one. Nothing for you to do." You can still take over with `T`.
- **Ready queue.** Two groups, **Unblocked** ("claimed in this order") and **Blocked**, capped at 16 rows with "Show 37
  more". The conversation for a blocked chunk says what it waits on, and if the blocker needs a human it offers **Open
  blocker C-6J9H**.
- **Landed · 48h.** Time and cost, with the retrospective one click away.

## Fleet: the team page

Support tools have an "agents and capacity" view: who is online, what each person is handling, and whether staffing
covers the queue. Desk's fleet page is that view for runners.

- **Header sentence.** "Both runners are online and every blizzard slot is busy. r-chatgpt declines new work until 14:41
  to stay on its OpenAI 7-day pace, leaving 2 slots idle after its deliveries land; 2 of r-claude's slots are parked on
  chunks that need you. The winter workspace has 4 free slots and nothing to claim." Tabs (Overview, Runners, Hosts,
  Workspaces, Enrollment tokens, Audit log) lead to the full tables.
- **Runner cards, presented as teammates.**
  - The top of each card shows the avatar with a presence dot, the harness and subscription, a state badge, and Pause
    `P` and ⋯.
  - Below that, one line says why the runner is in its state. For r-chatgpt: "Declining new work until 14:41. OpenAI 7d:
    8% spent, 7% elapsed — spend is ahead of the window's elapsed share, so it finishes what it holds and claims nothing
    new."
  - **Holding now** lists the chunks it holds as tiles, with red tiles for the ones that need you.
  - The footer shows landed today, spent today, agents, and a live last-seen counter.
- **Capacity · next 6 hours.** This is a staffing grid, the workforce-management chart from support tools. There is one
  column per half hour and one cell per slot, in five states: working, forecast busy, parked on you (red), idle while
  pacing (amber hatch) and free with nothing to claim (dashed). It shows what no other view does: two r-claude slots
  sit dead until a human acts, and two r-chatgpt slots sit idle until 14:41 while 12 unblocked chunks wait.
- **Workspaces & environment slots.** One row per workspace, with a tile per slot.
- **Right column.**
  - **Subscription windows.** The fill shows spend and the tick shows elapsed time, so fill past the tick means over
    pace. Each window has a sentence giving its pace and reset time.
  - **The host card for xps.** It shows a live last-report counter, the URL and the token, with Enroll, Revoke token
    (type to confirm) and **Retire host** (type `xps`).
  - **Fleet activity.**
- **Runner drawer** (`↵` or click). It explains the throttle, lists the profile, binds, the windows the runner spends and
  the chunks it holds, and offers Pause, Edit bindings and Retire runner (type the name).

## Mobile

On the phone the question is "does the fleet need me?", so the home tab is the inbox.

- **Bottom tab bar:** Needs you (red count), Work, Fleet (amber dot while a runner paces), Activity, More.
- **Needs you:**
  - A large title with its count, then the sentence ("Two chunks need you and hold back 5. The other 8 are running;
    today has cost ~$114.44.").
  - A segmented control: Waiting, Mine, Snoozed.
  - **Swipeable conversation cards.** Swipe right applies the suggested macro, and the ink underlay names it ("Take
    over"). Swipe left opens snooze. A hint line teaches both.
  - The **If you put the phone away** card ("Next needed: you, now. After that, ~13:45.") with Forecast and Go away.
  - Not waiting on you: in flight, ready, landed.
- **Conversation** is a full-screen push. The header shows the badge, a live waiting timer, the title and the key facts.
  The compact thread includes the bounce stack and the hub's message. A sticky composer has horizontally scrolling macro
  chips (the suggested one in ink), a note field and a send button that sends the suggested macro.
- **Macros, snooze and availability** open as bottom sheets. The availability sheet carries the full forecast, the
  notify toggles and **Go away — notify me**.
- **Work** has in flight, ready (four, then "Show all 53") and landed. Each row shows the title, id, stepper, cost and
  age.
- **Fleet** shows the runner cards with slot tiles and Pause/Details, a compact capacity grid with a "parked" row, the
  windows, the host with **Retire host** (an action sheet with typed confirmation) and the workspaces.

## Interaction model

**Keyboard (desktop).** Triage is single-key, as in Superhuman. Menus open with the first item focused and take
single-key accelerators, so `H` `2` snoozes for an hour and `;` `B` rebases and retries.

| Context      | Keys                                                                                                                                                                                                                                                                     |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Anywhere     | `⌘K`/`Ctrl K` palette · `W` walk-away forecast · `⇧A` away/on desk · `⇧D` compact · `⇧T` theme · `?` shortcuts                                                                                                                                                           |
| Go to        | `G I` Needs you · `G M` Mine · `G S` Snoozed · `G H` Handled · `G F` In flight · `G R` Ready · `G B` Backlog · `G D` Landed · `G L` Fleet                                                                                                                                 |
| Triage       | `J`/`K` (or arrows) next/previous, which opens the conversation · `E` resolve with the suggested macro · `T` take over · `R` requeue · `H` snooze · `A` assign · `I` assign to me · `;` or `M` macros · `N` write a note · `⌘↵` send note with the suggested macro · `Z` undo |
| Conversation | `P` open PR · `C` copy id · `Esc` leave the composer, then close the top layer                                                                                                                                                                                           |
| Fleet        | `J`/`K` move between runners · `↵` open the drawer · `P` pause/resume · `Esc` close                                                                                                                                                                                      |

The **palette** opens with the waiting items as *Suggested* actions ("Take over C-6J9H — holds back 5 · waiting 21m",
"Rebase & retry C-QWYS"). Next come actions on the open conversation, then chunk search by id, ref or title (`6j9h` and
`#1449` both work), then navigation, desk commands and preferences. Prefix `>` for commands only.

**Pointer.**

- The SLA pill, the presence rows, every icon button and each nav item show a tooltip with the shortcut.
- Clicking a dependency chip, a held-back chip or a slot tile jumps to that chunk's conversation.
- Summary fragments are links into their views.

**Touch.**

- Swipe right on a card applies the suggested macro, and swipe left snoozes.
- Tapping a card opens the conversation.
- Macro chips sit in thumb reach above the keyboard.
- Destructive actions use an action sheet with typed confirmation. Touch targets are at least 32 px, and 28 px in
  compact.

**Live.**

- Waiting timers and SLA rings tick every second, and an item turns into a filled red pill when it passes its target.
- In-flight previews update as reports arrive, and the item flashes once.
- The open running conversation shows a typing indicator and an elapsed timer.
- Host and runner last-seen counters count up.
- The forecast is labelled with its re-forecast time.

The typing dots are the only continuously animated element. They appear only in the open running conversation and
respect reduced motion.

**Undo, not confirm.** Every triage act (macro, snooze, assign, take over, pause) applies at once and offers Undo for
seven seconds (`Z`). Typed confirmation is reserved for acts that cannot be undone: cancel chunk, revoke token, retire
runner, retire host.

## Compact mode

`html[data-density="compact"]`, toggled with `⇧D`, the top-bar button or the palette, and kept in `desk.density`.

- **Rows and controls tighten.** Body text drops to 12 px, controls to 26 px, the top bar to 44 px and item padding to
  6 px.
- **Each queue item becomes two lines** (meta, then title). The reason and meta lines fold away for needs-you items,
  which keep their red glyph, red edge and SLA pill. Ready and landed items lose their preview line.
- **Fleet slot tiles become single-line chips** and the staffing grid's cells thin. On mobile, card titles clamp to one
  line and the forecast card drops its paragraph.

See `board-desktop-compact-breach.png` and `fleet-desktop-dark-compact.png`.

## Design tokens

Desk deliberately shares Console's foundations, so the two can merge:

- **Type.** Inter (variable, inlined woff2) at 400, 500 and 600, and a monospace stack only for ids, refs, URLs and
  versions.
- **Spacing and radii.** A 4 px spacing grid, and radii of 4, 6, 8 and 12.
- **Status roles.** `nh` red, `wait` amber, `run` blue, `dlv` teal, `done` green, `ready` grey, `backlog` dashed. They
  have the same values and the same glyph shapes as Console, so every glyph reads in greyscale.

The neutrals are a touch cooler and bluer than Console's (canvas `#f3f5f8`, sidebar `#eceff3`, text `#151920`). The cool
light theme is the default, it follows the OS, and dark is fully specified.

Desk adds four elements:

- **The SLA ring and pill.** The ring is red because the item needs a human, and it fills once the target is breached.
- **Avatars.** Neutral rounded squares for node agents and runners, and an ink circle for you.
- **The bounce stack.** A dashed amber-tinted container.
- **The staffing grid's hatched "idle while pacing" cell.** Amber, because pacing is a wait.

## Scaling from 8 running to 80

- **Needs you at 15.** The queue is the right shape for this. It stays oldest first, and the "Most held back" order puts
  the chain-blockers on top. The SLA pills show at a glance which items are over target. Filter chips split halts from
  asks and gates. With a team on the tenant, Unassigned and Mine divide the queue between people, which is what support
  tools are built for. Bulk select with `X` and a bulk macro would be the next feature.
- **In flight at 80.** The chips filter by runner, delivering and node. Order by latest activity floats the chunks that
  just reported, and the list virtualises. The presence bars in the nav become a proportional meter per runner ("62 /
  80"). The staffing grid groups slots by runner and draws one row per workspace.
- **Ready at 500.** It stays in claim order, capped, with unblocked first. Blocked chunks hide behind one chip, and search
  and the palette do the rest.
- **Fleet at 30 runners.** The teammate cards become the Runners table, sorted by attention: parked on you, declining,
  paused, working. The staffing grid becomes one row per runner, which is still readable, and the overview shows only
  runners that are not simply working.
- **Everything is URL-addressable** (`?queue=`, `?open=`), so a notification links straight into the conversation.

## What it deliberately drops from the baseline

- **The KPI strip.** It becomes the computed sentence. Spend sits in the sentence and the Live pill, and hub health is
  the pill.
- **The six equal kanban columns.** Needs-human is no longer a column. It is the inbox, and the other statuses are views.
  Desk has no board layout. Console keeps one, and Desk accepts the loss of the kanban's flow-at-a-glance in exchange for
  the staffing grid and the live previews.
- **The right rail.**
  - The runner meters become the presence rows and the fleet page.
  - Asks and gates join the inbox.
  - The activity feed becomes live previews in In flight, the fleet activity card and the Activity view.
- **Chunk detail as tabs** (general, node history, artifacts, transcripts). The history is the conversation itself, and
  the properties are its sidebar.
- **One monospace face in all caps, and decorative colour.** As in Console.

## Risks and what I am least sure of

- **The inbox metaphor fits halts better than gates.** A bounce-cap halt really is a conversation: the hub explains, you
  answer with an action. An open gate is closer to an approval. It fits the queue, but its "thread" is a diff and a
  decision, not a dialogue. There were no open asks or gates in the capture, so they are only counted (0), not drawn.
- **Notes that travel with macros are a new capability.** Requeue-with-guidance is the most support-like feature here,
  and I believe the most valuable, but blizzard has to carry an operator note to the next agent for it to be real.
- **SLA can feel like pressure.** In support work an SLA is a promise to a customer. Here it protects throughput: a
  waiting chunk holds others back. The 30-minute target is a placeholder, and it should probably scale with what the item
  holds back.
- **Undo instead of confirm** depends on the hub accepting a short delay or a reversal for requeue and take-over.
- **The forecast must be honest.** Like Tacit's, it loses trust the first time it is badly wrong. Desk hedges with `~`,
  with "estimated from 7 days of node durations", and by keeping the raw state one click away. But the "next likely
  needed" claim has to be earned by the model behind it.
- **A familiar shape may read as generic.** It may look like "a helpdesk skin" in a PNG. The craft is in auto-advance,
  in undo, and in the clock stopping when you take an item.
- **"Holds back" uses Console's definition:** 5 (2 directly), counting only the edges known by id. The sidebar shows both
  numbers.
- **Mock data.**
  - The capture has the needs-human chunks, the in-flight set, the queues, the fleet and the windows.
  - C-QWYS's node history and escalation reason are reconstructed from the activity feed.
  - The tokens and costs on C-6J9H's last eight visits, C-QWYS's usage, the runners' landed-today and spent-today splits
    (which do sum to 18 and ~$114.44) and the live preview lines are illustrative.
  - The forecast's times and amounts are illustrative.
  - Saved views, the order menu, Activity, Usage, Settings and the fleet tabs other than Overview are stubs that answer
    with a toast.

## Files

| File                               | What                                                                    |
| ---------------------------------- | ----------------------------------------------------------------------- |
| `board-desktop.html` / `.png`      | Needs you, C-6J9H open, light, comfortable (1600×1000)                  |
| `fleet-desktop.html` / `.png`      | Fleet overview: teammates, staffing grid, windows, host (1600×1000)     |
| `board-mobile.html` / `.png`       | Needs you on mobile (390×844 @2x)                                       |
| `fleet-mobile.html` / `.png`       | Fleet on mobile (390×844 @2x)                                           |
| `board-desktop-macros.png`         | Macro picker with descriptions, a note written in the composer          |
| `board-desktop-snooze.png`         | Snooze menu with its "the clock keeps running" warning                  |
| `board-desktop-takeover.png`       | Take-over sheet with the command to copy                                |
| `board-desktop-mine.png`           | C-6J9H in Mine after take-over: SLA met, Hand back as the primary       |
| `board-desktop-zero.png`           | Inbox zero, with the full walk-away forecast and notify toggles         |
| `board-desktop-forecast.png`       | "Walk away?" pop-over while two items still wait                        |
| `board-desktop-flight.png`         | In flight: live previews, C-WSCR's full history and typing indicator    |
| `board-desktop-ready.png`          | Ready queue, blocked C-ZEVE pointing at its needs-human blocker         |
| `board-desktop-dark-bounces.png`   | Dark theme, C-6J9H's bounce stack expanded                              |
| `board-desktop-compact-breach.png` | Compact density at 13:21, with C-6J9H past its 30-minute target         |
| `board-desktop-palette.png`        | ⌘K palette with the suggested resolutions                               |
| `fleet-desktop-drawer.png`         | Runner drawer on r-chatgpt (declining)                                  |
| `fleet-desktop-dark-compact.png`   | Fleet in dark, compact, keyboard cursor on r-claude                     |
| `fleet-desktop-retire-confirm.png` | Type-to-confirm on Retire host                                          |
| `board-mobile-thread.png`          | C-6J9H's conversation with macro chips and composer                     |
| `board-mobile-swipe.png`           | Swiping C-6J9H right to take over                                       |
| `board-mobile-macros.png`          | Macro sheet for C-QWYS                                                  |
| `board-mobile-away.png`            | "If you put the phone away" sheet with notify toggles                   |
| `board-mobile-zero.png`            | Mobile inbox zero                                                       |
| `board-mobile-work-dark.png`       | Work tab in dark                                                        |
| `board-mobile-compact.png`         | Mobile compact                                                          |
| `fleet-mobile-retire-confirm.png`  | Retire action sheet with typed confirmation                             |

Every mock is a single offline file (inline CSS, JS and an inlined Inter woff2, with no network fetches) and accepts URL
state for review:

- **Every mock:**
  - `?theme=light|dark`
  - `?density=compact`
  - `?at=HH:MM:SS` sets the mock clock, for example `13:21:30` to show an SLA breach
- **`board-desktop.html`:**
  - `?queue=needs|mine|snoozed|handled|flight|ready|backlog|done`
  - `?open=C-6J9H`
  - `?zero=1` resolves both waiting items
  - `?mine=C-6J9H` marks it taken over
  - `?expand=C-6J9H` opens its bounce stack
  - `?macros=1`, `?snooze=1`, `?take=1`, `?palette=1`, `?forecast=1`, `?keys=1`
  - `?note=…` pre-fills the composer
  - `?cursor=C-…`
- **`fleet-desktop.html`:** `?drawer=r-chatgpt|r-claude`, `?confirm=retire`, `?cursor=r-claude`, `?palette=1`
- **`board-mobile.html`:**
  - `?tab=inbox|work`
  - `?seg=needs|mine|snoozed`
  - `?open=C-6J9H`
  - `?sheet=macros|snooze|away`
  - `?swipe=C-6J9H&dir=right|left`
  - `?zero=1`
- **`fleet-mobile.html`:** `?confirm=retire`, `?sheet=away`
