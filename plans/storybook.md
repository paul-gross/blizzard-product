---
epic: storybook
refinement: refined
status: horizon
---

# Plan — `epic:storybook`

The board is assembled from a kit of its own parts — buttons, chips, menus, dialogs, tabs, the select row and the
skeleton — and nobody can see them together. To look at a dialog today, the product owner finds a screen that happens to
open one, with whatever data the fleet happens to hold that day, and judges the part through the page wrapped around it.
A variant nobody's screen exercises cannot be seen at all, and feedback on a part arrives as feedback on a page. This
epic gives the kit a place of its own: a Storybook where every part stands alone in each of its states, with controls to
push on it, so the owner can browse what exists, play with it, and say what should change before the next surface is
built from it.

## What to build

- **A Storybook over the fleet kit.** Every reusable part in the `fleet` library's kit has stories that show it in each
  meaningful state — empty, loading, disabled, overflowing, and at the narrow widths the board supports — with controls
  that let a reader change its inputs and watch it answer. The kit comes first and whole.
- **Then the fleet library's composed pieces.** Once the kit is in, the pieces the `fleet` library composes from it and
  shares between the hub's board and the runner's panel — the chunk timeline and its facts, escalation, delivery and
  token breakdown, the artifact views, the chunk page header and tabs, the transcript viewer, the board header and the
  mobile chrome — get stories of their own, so a piece can be judged in states the dogfood fleet rarely holds. The hub
  shell's project lens selector joins them once it takes its projects from the shell rather than fetching them itself.
  Components that belong to one app's pages stay out: they are judged on the page they serve.
- **Shared builders, separate states.** The composed pieces need data that stands in for a live fleet. The `fleet`
  library's testing entry gains builders that produce each wire shape with realistic defaults, typed against the
  generated client, and stories and specs each compose the states they need from them. A spec keeps spelling out exactly
  what its assertion depends on, so tuning a story's look can never quietly weaken a test, and a change to the wire
  breaks both sides at compile time in one place. Specs move onto the builders as they are next touched, not in a sweep.
- **One command, from any checkout.** The workshop runs locally: a single script in the web workspace starts it, in the
  source checkout or any feature environment, against the kit as that checkout holds it. Nothing is published or
  deployed.
- **The workspace that builds blizzard comes first.** Blizzard is built by agents working in its own winter workspace,
  so the workshop is not finished until that workspace serves it. The workspace declares it as a service beside the two
  app dev servers, on a port of its own in every feature environment's band, so `winter service up` starts it in every
  environment and an agent finds it at a known address rather than standing it up by hand. The verification method's
  setup points there. This lands with the epic, not after it: a workshop its builders cannot reach is a workshop nobody
  uses.
- **A story with every new part, held by the build.** Adding a part to the kit includes its stories. The web workspace's
  structural gate — the same gate that already holds the kit floor — fails when a kit component has no stories beside
  it, and a part that genuinely cannot have them is named in the gate's exemptions with its reason. The gate holds the
  kit and nothing else. A rule beside the kit floor in `blizzard-context`'s frontend structure says so, so an agent
  adding a part knows before the gate tells it, and the rule extends the expectation to a composed piece only where that
  piece lives in a dedicated shared area — a home for parts many features use, never the domain folder of one feature. A
  piece inside a feature's own domain may have stories, but no rule asks for them. The workshop stays a whole picture of
  the kit rather than a snapshot of the week it was built.
- **A way of working, written into the verification matrix.** `blizzard-context`'s verifiability matrix gains a manual
  `web:` method for the workshop: a person, or a frontend-verifier agent, starts it in the environment under change,
  opens the stories of every part the change touches, and works their states and controls at the narrow widths. Its
  guidance makes the workshop part of how frontend work is done, not only how it is checked. An agent build that changes
  anything a person sees on the board or the panel uses the workshop while it builds — finding the part to reuse before
  inventing another, and shaping a change against the part's stories — and inspects every part it touched there before
  it calls the work done.

This epic lands before `epic:ui-toolkit`. The workshop is where that survey's candidates are judged, side by side with
the parts they would replace, and whatever it adopts arrives in the workshop rather than beside it.

Stories are for people in this epic, with one exception that protects them. The gate renders every story and fails on
one that throws or logs an error while it renders, because a workshop whose stories have quietly broken is one its
builders stop trusting, and the manual method trusts it. Stories assert nothing about behavior or appearance:
interaction tests would repeat what the component specs and the browser-rendered layout checks already prove, and
screenshot comparison asks for stored baselines and a review flow this local workshop has no use for. Once the shared
builders and the stories both exist, a spec may take a story as its fixture, so the two describe a state once rather
than twice; whether specs should is a question for after the owner has lived with them.

## Toolchain

The workshop rides Storybook's Vite-based Angular framework, pinned to an exact version, with its configuration written
by hand beside the `fleet` library rather than generated by Storybook's installer, which brings test addons, a second
build system, and choices this plan has already made. That framework builds the way the workspace's own Angular build
does, runs zoneless as the apps do, and is given the same global stylesheets, so a part in the workshop looks as it does
on the board. The render check builds the workshop and opens each story in a browser; it stays out of the unit-test
builder, so stories never become a second test suite by accident.

Two costs come with it, accepted with open eyes. Every Angular major waits for a Storybook release that admits it —
historically a week to two months — so the workshop is one more thing an Angular upgrade carries. And the framework is
still in preview, so its options may move when it settles.

Storybook's peer dependencies collide with the workspace's exact Angular and test-runner versions, and a plain install
refuses them. The first piece of work finds overrides that hold under a strict install; only if none do does the
workspace relax peer checking, and only with the owner's sign-off, since that relaxation reaches every install in CI.

The board has one theme today, so the workshop shows that one. A second theme is design work of its own; when it
arrives, the workshop gains the switch to show both.
