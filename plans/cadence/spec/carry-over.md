# Carry-over

The epic lands on a hub whose only way to run a routine is a person asking it to, and whose every run rests behind the
promote gate until a person admits it. It leaves a hub that keeps a project's rhythms itself: each cadence mints its due
errands within a minute of their moment, promoted unless a chore holds them, one live run per errand, each run carrying
the mark of the errand that minted it. Nothing deployed changes id, no run started by hand behaves differently, and no
runner needs upgrading.

## The order of landings

Each piece lands on its own, leaves `master` green, and is proven by the methods it names. A piece that changes
`blizzard-context` lands that half there, proven by `blizzard-context:reference-lint`,
`blizzard-context:markdown-format`, `blizzard-context:markdown-lint`, and `blizzard-context:registry-drift`, and no half
cites a rule or method that has not landed. Every piece that changes an operator surface regenerates its contract
documents and carries its change note under `changes/unreleased/` (`blizzard:change-notes`). No piece changes the fleet
wire, so none extends `blizzard-mock`.

1. **The routine's slug.** Waits for nothing. The additive revision giving every routine a `slug` filled from its
   `name`, with the project's uniqueness moved onto it; `slug` on the routine's views, its create, and its config entry;
   `name` editable; the CLI resolving by slug; run titles and charges showing the display name
   ([routine-slug.md](./routine-slug.md)). The routines-and-scopes and config-documents pages. Its context half:
   `domain/routines-and-scopes.md` names the slug as the lineage. Proven by `blizzard:unit-test` over a create with only
   `name`, with both, and with an invalid slug refused; `blizzard:component-test` over the migration on a store whose
   routine names include one that is not a valid slug, its findings, baselines, and runs still read by the routine
   afterwards, and an apply of a document keyed by `name` alone leaving the routine unchanged; `blizzard:compat` against
   every supported release's client; `blizzard:store-contract`, `blizzard:cli-contract`, and `web:client-drift`.
2. **The record.** Waits for piece 1, since a chore links its routine by slug. `cronsim` and `tzdata` as runtime
   dependencies, and the schedule module that alone reaches the cron library. The four cadence tables in one additive
   revision and its migration test; `cad_` and `chr_` ids; `RecordKind.CADENCE`; `hub/domain/cadence/` with the model,
   its validation, and the six verbs, and its row in `_DOMAIN_PACKAGE_LAYERS`; the store adapter and its census entries;
   the routes, with `chunk:control` asked of a promoting write; the `cadences` section of the config document, its
   schema, its reconcile after routines, and its export; `attention` on the view; the `cadence` CLI group with its chore
   verbs ([record.md](./record.md), [surfaces.md](./surfaces.md) §Needs attention, §The CLI). The `cadences` flag is
   declared, off. `docs/deployment/cadences.md` with the record, its fields, and attention, marked as minting nothing
   while the flag is off; `config-documents.md`'s section. Its context half: `domain/cadences.md` with the cadence, the
   chore, and the errand, routed from `domain/index.md`. Proven by `blizzard:unit-test` over every validation refusal,
   among them a six-field and a never-firing cron expression, and over every attention reason, including a listed scope
   unlinked beside one still linked and an `all` chore whose last scope is retired; `blizzard:component-test` over the
   store and over an apply that declares a routine and its cadence in one document, an apply naming an unknown routine
   accepted with `routine_unknown`, an edit that keeps a chore's key keeping its id, and one that drops and re-adds a
   key minting a new one; `blizzard:store-contract`, `blizzard:cli-contract`, `blizzard:compat`, and `web:client-drift`.
3. **The mint.** Waits for piece 2, and for the clock fixture of `epic:test-suite/controlled-clock`, which its service
   tests run on. The additive revision adding `cadence_id` and `chore_id` to `work_item_runs` with its index; the mark
   on `RunContext` and `RunIdentity`; `WorkItemAuthor.cadence`; `RunService.run`'s mark and promote arguments, and
   `create_run_with_chunk` writing the mark and, for a promoting chore, the promote fact and the tail position in the
   same transaction; `ChunkFacts.started_at()`; the due rule; the `cadence` sweep, its interval, its roster entry, its
   pass bound, its locked write, and its entry in `tests/pass_isolation.py`, minting only while the flag is on
   ([due.md](./due.md), [mint.md](./mint.md)). `observability.md`'s row. Its context half: `domain/cadences.md` gains
   when an errand is due and `bzh:errand-one-live-run`; `domain/work/statuses.md` and `domain/routines-and-scopes.md`
   name the promoting chore; `architecture/crash-correctness/hub.md` gains §The cadence mint. Proven by
   `blizzard:unit-test` over the due rule at every edge — a spacing alone with and without a last run, a schedule alone
   with the latest moment before and after the reference, both with eligibility falling between two moments, each anchor
   including a run stopped before it started, a live last run and one in `needs_human` holding the errand, a stopped one
   releasing it, a hub away for a week minting each errand once, a never-run errand of a chore that has minted borrowing
   its sibling's reference and one of a chore that never has borrowing its creation, a cron schedule's moments across
   both daylight-saving changes, read as Debian's cron reads them,, and a chore edited to another routine keeping its
   errands' last runs; `blizzard:component-test` over the sweep with two tenants and a poisoned third, a pass bound
   leaving the rest to the next pass, a hand-started run of the same routine and scope neither counting as the errand's
   run nor holding it back, a refused errand beside a minted one, and two passes racing one errand minting once;
   `blizzard:postgres-uow-concurrency` over the new lock key; `blizzard:service-test` on the controlled clock
   ([§Proven on the controlled clock](#proven-on-the-controlled-clock)) with the flag on, moving the clock past a moment
   and stepping the pass to find one run per errand, promoted or resting as each chore says, the delta settling to
   `full` with no baseline, a clock moved through three missed moments minting each errand once, and a spacing of ten
   days proven by moving the clock ten days rather than waiting; `blizzard:store-contract`; `blizzard:compat`; and
   `blizzard:manual-hub` with a real cadence firing on a minute's moment.
4. **Errands in view.** Waits for piece 3. The `errands` route, its views and their `state` and `due_at`; the counts on
   the cadence's view; the `cadence` on a run's views; `blizzard hub cadence errands` and the counts on `show`
   ([surfaces.md](./surfaces.md) §Errands, §Runs). `docs/deployment/cadences.md` gains errands, when they are due, and
   the anchors. Proven by `blizzard:unit-test` over every state and over `due_at` under each check;
   `blizzard:service-test` on the controlled clock reading a cadence's errands before and after a pass mints them, after
   its runs settle, and with `due_at` matching the instant the clock is then moved to; `blizzard:cli-contract`;
   `blizzard:compat`; and `web:client-drift`.
5. **The board.** Waits for pieces 2 and 4. The Cadences noun in the gardening tab, its list and detail, their
   mutations, the *Kept by* line on a routine's detail, and the cadence on a run's detail ([surfaces.md](./surfaces.md)
   §The board). `docs/deployment/board.md`. Proven by `web:unit-test`, `web:lint`, `web:typecheck`, `web:story-render`,
   `blizzard:ui` on the controlled clock, whose fixture fixes the browser's clock with the hub's, over creating a
   cadence, adding a chore over all scopes, and reading its errands with one stopped and its next moments as the clock
   moves, and `web:shell-sweep` on a narrow screen.
6. **The existing three, moved over.** Waits for piece 5. `docs/deployment/cadences.md` gains worked cadences for the
   three epics that wait on this one: the mutation-testing corpus pass over its four scopes, the ideation routine, and a
   gardening routine whose trend wants a steady series, each in delta mode where its routine converges.
   `docs/deployment/routines-and-scopes.md` no longer says a routine runs only when someone starts it. The release that
   carries this piece turns `cadences` on, saying so in its notes. In the workspace, which this epic's pieces do not
   otherwise touch, `workspace:/context/project/workflows/tending-the-garden.md` learns that a routine a cadence keeps
   is not a routine to start by hand. Proven by `blizzard:flag-lifecycle` and `blizzard:manual-hub` with the flag on by
   default.

The epic is delivered when piece 6 lands. Its one follow-up removal is the routine's: a config entry keyed by `name`
alone, and a create sending `name` without `slug`, are deprecated by piece 1, and their removal is a standalone
`(BREAK)` issue that waits until an operator decides a release may carry it. A run started by hand stays exactly what it
was. The `cadences` flag is retired about one stable release after it turns on, as every flag is.

## Proven on the controlled clock

Every claim this epic makes about time is proven by moving a clock, never by waiting for one. Every test that judges
when an errand is due, mints it, or shows when it comes due next runs on the controlled clock of
`epic:test-suite/controlled-clock`: a service or UI spec asks for it with `test.use({ clockMode: 'controlled' })` at
`fresh-hub`, moves time with the fixture's `clock.advance` and `clock.set`, and then steps the cadence sweep through its
pass route, since a controlled stack refuses a timed sweep and nothing moves until the test asks. A spec reads the
instant it asserts against from the fixture's `clock.now()`, never from the machine's clock, which the fixture's own
check refuses. A ten-day spacing is ten days of `advance`, a hub away for a week is a week of it with no pass between,
and a moment across a daylight-saving change is the clock set to either side of it.

That holds only if the cadence keeps time through the hub's injected clock and nothing else (`bzh:injected-clock`). The
pass reads `now` from it once and hands that instant to the due rule, the cron library is always given an instant to
iterate from and never asked for the time, and an anchor's instants come from the run's facts, stamped by the same
clock. The sweep's own sixty-second timer is mechanical and stays on real time, which is why a test steps the pass
rather than waiting for it. The component tier keeps its in-process virtual clock, as the controlled-clock epic leaves
it.

## Hub and runners on different versions

A runner sees nothing of a cadence. It claims a cadence's run as it claims any chunk of its project, so a runner of any
supported release works against a hub carrying every piece. A board older than piece 4 shows a cadence's run as a run,
and ignores the `cadence` its view gains.

## What an operator does once

Each piece's revision is applied with `blizzard hub migrate`, as every hub revision is (`bzh:manual-migrations`);
nothing else is owed. An operator whose fleet already starts a routine by hand on a rhythm declares a cadence for it,
and stops starting it by hand, once the flag is on.

## Left for later

A chore's urgency is `epic:urgency`'s: it adds one chore field, and a cadence's run takes the urgency its chore names. A
cadence that waits for last week's findings to be dealt with before minting again needs a condition on open findings,
which the plan leaves for later. The release train and the store fold wait for a routine that can carry them, since no
fleet chunk's environment holds the workspace where their procedures live. The garden's delivered plans, which foresaw a
cadence as a field on the routine and a routine with no next-run time, stay frozen as they shipped; this plan is the
intent that replaced them.
