# Plan — `epic:demo`

Everything blizzard does begins with a credential. A newcomer who wants to see what a fleet actually does must first
obtain a coding-harness subscription, a forge token, and a repository worth pointing at — which is a great deal of trust
to ask of someone who has not yet seen the product work. A graph author has a quieter version of the same problem: the
only way to find out what their graph does is to spend a night and some money finding out. This epic makes blizzard
runnable with nothing real attached.

The mock fleet already exists for blizzard's own tests, with a mock for every seam. What it lacks is three things.

## What to build

- **A harness that answers plausibly.** Today's mocks execute scripted prompts; a demo needs one that responds to a real
  prose prompt in a way that looks like work, deterministically and at no cost.
- **A mock forge holding a backlog.** Items to browse, ingest, and land against, so the whole path from intake to
  delivery is walkable.
- **Demo as a mode of blizzard.** The real hub, runner, graphs, and board, above mock seams — packaged as a way to run
  blizzard rather than as a sibling checkout that will drift.
- **Named scenarios that play out over time.** A good night, a chunk that wedges, a question that needs answering — so a
  newcomer sees the fleet's interesting moments in minutes instead of waiting for one to happen.

## Open questions

- How plausible the mock harness must be to be worth having, and whether plausibility requires a model at all or is
  better served by something wholly deterministic.
- Whether the mode is selected by configuration or is its own command, and what stops someone running it by accident
  against real credentials.
- What keeps the scenarios honest as the graph evolves, since a demo that shows last quarter's fleet is worse than no
  demo.
