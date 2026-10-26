# Redesign brief

The brief every redesign proposal under this folder answers to.

## What exists

The board at `blizzard.grosscode.net` is blizzard's first design, captured in [baseline/](./baseline/):

- `desktop-board.png`: the chunk board. A KPI strip runs across the top (chunks, backlog, ready, waiting, needs-human,
  running, done, today and yesterday spend, hub health). Below it is a tab row (board, fleet, graphs, events, gardening,
  admin) and a project lens. Six kanban columns (backlog, ready, wait/human, needs-human, running, done-48h) hold cards
  that show work ref, title, blocking chain, node, chunk short id, cost and PR link. A right rail carries runner
  subscription meters, open questions, open gates, and the activity feed.
- `desktop-fleet.png`, `desktop-fleet-full.png`: fleet. There is a host list and a host detail. The detail carries
  connection info, workspaces with environment slots (chunk to node to runner), subscription-usage window meters
  (Anthropic 5h/7d, OpenAI 7d), and runners (binds, spends, harness, gates, version, held chunks, pause).
- `desktop-chunk.png`, `mobile-chunk-full.png`: chunk detail, with tabs for general, node history (build to review to
  pre-push to deliver bounces, with tokens and cost per step), artifacts and transcripts.
- `mobile-board.png`, `mobile-fleet.png`: the mobile shell. It has a triage-first card list led by needs-you, running,
  ready, done-today and runners-up counts.
- `*.txt` and `chunks-sample.json` hold the real text and data behind those screens. Use them, not lorem ipsum.

Its virtues are worth keeping. It is information-dense, everything is on one screen, it is honest about cost, and it
shows live movement. Its faults are the reason for this epic. It uses one monospace face at one weight in all caps. Its
hierarchy is weak: needs-human weighs the same as done. The 53 ready cards bury the 2 that need a human. Colour is used
as decoration rather than signal, and the layout is a 2024 dark dashboard.

## Who it is for

`persona:application-architect` (see `charter/personas/application-architect.md`) and `persona:harness-engineer`. These
are operators: power users who queue work, walk away, and return to see what landed, what is stuck and what it cost.
They read the board as an instrument, not a brochure. They want a keyboard first, density, speed, and truth at a glance.
When something goes wrong they need to take it over in one step. On mobile the operator is usually away from the desk
and answering "does the fleet need me?"

## The domain, in brief

- **Chunk**: one unit of work (one or more issues), walking a **graph** of **nodes** (triage, build, review, mutation,
  pre-push, deliver). A failure **bounces** it back to an earlier node.
- Statuses: backlog, ready (often blocked by other chunks, forming chains), waiting, needs_human, running, delivering,
  done.
- **Runner**: a worker pool on a **host**, bound to workspaces with N environment slots and one harness (claude code,
  opencode). Runners spend a subscription (Anthropic, OpenAI) whose rolling windows (5h, 7d) throttle work.
- **Gates**: points where a human decides. **Asks**: questions an agent puts to a human.
- Everything streams live, and cost accrues per step.

## Deliverables

Put everything in your own subfolder, `artifacts/<your-slug>/`:

1. `README.md`: the concept. Give the thesis in a paragraph, then the principles, the interaction model (keyboard,
   pointer, touch), how it scales from 8 running to 80, what it deliberately drops from the baseline, and the risks.
2. Four self-contained HTML mocks using real data from the baseline: `board-desktop.html` (1600x1000),
   `fleet-desktop.html` (1600x1000), `board-mobile.html` (390x844) and `fleet-mobile.html` (390x844). Each is a single
   file with inline CSS and JS, and no network fetches. Web fonts may be inlined or system/variable fonts named, but the
   mocks must render offline. Interactions that carry the concept (keyboard nav, command palette, hover states, live
   ticking) should actually work.
3. A PNG of each mock, rendered with headless Chrome:
   `google-chrome --headless=new --disable-gpu --hide-scrollbars --window-size=1600,1000 --screenshot=<out>.png file://<abs path>`
   (use 390,844 with `--force-device-scale-factor=2` for mobile). Look at your own screenshots and iterate until they
   are right. Do not use the chrome-devtools MCP: other designers share that browser.

Do not touch anything outside your subfolder.
