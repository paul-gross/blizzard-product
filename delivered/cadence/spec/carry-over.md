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
documents and carries its change note (`blizzard:change-notes`). No piece changes the fleet wire, so none extends
`blizzard-mock`.

1. **The routine's slug.** Waits for nothing. The additive revision giving every routine a `slug` filled from its
   `name`, with the project's uniqueness moved onto it; `slug` on the routine's views, its create, and its config entry;
   `name` editable; the CLI resolving by slug; the board joining a routine's pages and runs by slug and showing its
   name; run titles and charges showing the display name ([routine-slug.md](./routine-slug.md)). The operator pages on
   routines and scopes and on config documents. Its context half: the domain's routines-and-scopes spoke names the slug
   as the lineage. Proven by `blizzard:unit-test` over a create with only `name`, with both, and with an invalid slug
   refused; `blizzard:component-test` over the migration on a store whose routine names include one that is not a valid
   slug, its findings, baselines, and runs still read by the routine afterwards, an apply of a document keyed by `name`
   alone leaving the routine unchanged, and one re-applied after an operator edited the display name leaving that name
   alone; `web:unit-test` over a routine whose name and slug differ; `blizzard:compat` against every supported release's
   client; `blizzard:store-contract`, `blizzard:cli-contract`, and `web:client-drift`.
2. **The record.** Waits for piece 1, since a chore links its routine by slug. `cronsim` and `tzdata` as runtime
   dependencies, and the schedule module that alone reaches the cron library; the four cadence tables in one additive
   revision and its migration test; the `cad_` and `chr_` ids; the `cadence` record kind; the cadence package with its
   model, its validation, and the six verbs, declared in the domain's layering; the store adapter and its read-census
   entries; the routes, with `chunk:control` asked of a promoting write; the `cadences` section of the config document,
   its schema, its reconcile after routines, and its export; `attention` on the view; the `cadence` CLI group with its
   chore verbs ([record.md](./record.md), [surfaces.md](./surfaces.md) §Needs attention, §The CLI). The `cadences` flag
   is declared, off. The operator page on cadences, with the record, its fields, and attention, marked as minting
   nothing while the flag is off; the config-documents page's section. Its context half: a new domain spoke for the
   cadence, the chore, and the errand, routed from the domain's index. Proven by `blizzard:unit-test` over every
   validation refusal, among them a six-field and a never-firing cron expression, over the schedule module at every edge
   it pins — a moment exactly at the instant asked, both daylight-saving changes for a fixed-hour and an every-hour
   expression, and the day-of-month or day-of-week match — and over every attention reason, including a listed scope
   unlinked beside one still linked and an `all` chore whose last scope is retired; `blizzard:component-test` over the
   store and over an apply that declares a routine and its cadence in one document, an apply naming an unknown routine
   accepted with `routine_unknown`, an edit that keeps a chore's key keeping its id, and one that drops and re-adds a
   key minting a new one; `blizzard:flag-lifecycle`; `blizzard:store-contract`, `blizzard:cli-contract`,
   `blizzard:compat`, and `web:client-drift`.
3. **The mint.** Waits for piece 2, and for the clock fixture of `epic:test-suite/controlled-clock`, which its service
   tests run on. The additive revision adding the errand's mark and its index to the run's identity, and `linked_at` to
   the routine's scope link; the mark on the run's identity as the hub reads it; the `cadence` author kind; the chunk's
   derived start; the due rule; the mint's unit of work with its lock order; the `cadence` sweep, its interval, its boot
   pass, its roster entry, its pass bound, and its place in the poisoned-tenant fixture, minting only while the flag is
   on ([due.md](./due.md), [mint.md](./mint.md)). The test harness learns the new sweep in the same change: every
   harness profile sets `[sweeps.cadence] mode = "stepped"`, and the harness's lists of steppable sweeps, hub-wide and
   per tenant, gain `cadence`, so no profile fails its stepped check and a spec can step the pass. The observability
   page's row. Its context half: the cadence spoke gains when an errand is due and `bzh:errand-one-live-run`; the
   domain's statement of the promote gate, in the chunk's statuses and the routine's run, names the promoting chore; the
   hub's crash-correctness record gains the cadence mint. Proven by `blizzard:unit-test` over the due rule at every edge
   — a spacing alone with and without a last run, a schedule alone with the latest moment before and after the
   reference, both with eligibility falling between two moments, each anchor including a run stopped before it started,
   a live last run and one in `needs_human` holding the errand, a stopped one releasing it, a deleted one settling as
   withdrawn and not re-minted, a grouped one followed into its survivor and holding its errand until the survivor
   settles, several errands' runs grouped into one chunk each holding its own errand, a moment missed across a restart
   minting once on the first pass after it, a week away minting each errand once, a never-run errand counting from when
   it joined under each kind of chore and under a spacing beside its schedule, and a chore edited to another routine
   keeping its errands' last runs; `blizzard:component-test` over the sweep with two tenants and a poisoned third, a
   pass bound leaving the rest to the next pass, a hand-started run of the same routine and scope neither counting as
   the errand's run nor holding it back, a refused errand beside a minted one, two passes racing one errand minting
   once, and a promoting mint racing an operator's promote landing each at its own position;
   `blizzard:postgres-uow-concurrency` over the new lock order; `blizzard:service-test` on the controlled clock
   ([§Proven on the controlled clock](#proven-on-the-controlled-clock)) with the flag on; `blizzard:harness-check`;
   `blizzard:config-contract` for the sweep's mode key; `blizzard:store-contract`; `blizzard:compat`; `web:client-drift`
   for the pass route's regenerated description of the sweeps it serves; and `blizzard:manual-hub` with a real cadence
   firing on a minute's moment.
4. **Errands in view.** Waits for piece 3. The `errands` route, its views and their `state` and `due_at`; the counts on
   the cadence's view; the `cadence` on a run's views; `blizzard hub cadence errands` and the counts on `show`
   ([surfaces.md](./surfaces.md) §Errands, §Runs). The operator page gains errands, when they are due, a missed moment,
   and the anchors. Proven by `blizzard:unit-test` over every state and over `due_at` under each check;
   `blizzard:service-test` on the controlled clock reading a cadence's errands before and after a pass mints them, after
   its runs settle, and with `due_at` matching the instant the clock is then moved to; `blizzard:cli-contract`;
   `blizzard:compat`; and `web:client-drift`.
5. **The board.** Waits for pieces 2 and 4. The Cadences noun in the gardening tab, its list and detail, their
   mutations, the *Kept by* line on a routine's detail, and the cadence on a run's detail ([surfaces.md](./surfaces.md)
   §The board). The board's operator page. Proven by `web:unit-test`, `web:lint`, `web:typecheck`, `web:story-render`,
   `blizzard:ui` on the controlled clock, whose fixture fixes the browser's clock with the hub's, over creating a
   cadence, adding a chore over all scopes, and reading its errands with one stopped and its next moments as the clock
   moves, and `web:shell-sweep` on a narrow screen.
6. **The existing three, moved over.** Waits for piece 5. The operator page on cadences gains worked cadences for the
   three epics that wait on this one: the mutation-testing corpus pass over its four scopes, the ideation routine, and a
   gardening routine whose trend wants a steady series, each in delta mode where its routine converges. The page on
   routines and scopes no longer says a routine runs only when someone starts it. The release that carries this piece
   turns `cadences` on, saying so in its notes. In the workspace, which this epic's pieces do not otherwise touch, the
   tending-the-garden workflow learns that a routine a cadence keeps is not a routine to start by hand. Proven by
   `blizzard:flag-lifecycle` and `blizzard:manual-hub` with the flag on by default.

The epic is delivered when piece 6 lands. Its one follow-up removal is the routine's: a config entry keyed by `name`
alone, and a create sending `name` without `slug`, are deprecated by piece 1, and their removal is a standalone
`(BREAK)` issue that waits until an operator decides a release may carry it. A run started by hand stays exactly what it
was. The `cadences` flag is retired about one stable release after it turns on, as every flag is.

## Proven on the controlled clock

Every claim this epic makes about time is proven by moving a clock, never by waiting for one. A test that judges when an
errand is due, mints it, or shows when it comes due next runs, advances the clock, and asserts, and nothing in it
sleeps.

The pure tiers need no daemon at all. The due rule and the schedule module are functions of an instant they are handed,
so a unit test hands them instants: a moment either side of a daylight-saving change, a week of missed moments, a run
that finished nine days and twenty-three hours ago. The component tier keeps its in-process virtual clock, as the
controlled-clock epic leaves it.

The service and UI tiers run on the controlled clock of `epic:test-suite/controlled-clock`. A spec asks for it with
`test.use({ clockMode: 'controlled' })` at `fresh-hub`, moves time with the fixture's `clock.advance` and `clock.set`,
and then steps the cadence sweep through its pass route: a controlled stack refuses a timed sweep, so no pass runs until
the spec asks for one. It reads the instant it asserts against from the fixture's `clock.now()`, never from the
machine's clock, which the fixture's own check refuses. Each timing the spec promises is one such move:

- **A moment.** Set the clock to 18:59, step the pass and see nothing minted, advance one minute, step, and see one run
  per errand, promoted or resting as each chore says.
- **A restart across the moment.** Set the clock to 18:59, advance to 19:05 with no pass between, which is what a hub
  down across 19:00 sees, then step once and see the 19:00 run minted, once. Stepping again mints nothing.
- **No catch-up.** Advance three days without a pass, step, and see one run per errand.
- **A spacing.** Settle a run, advance nine days and twenty-three hours and see nothing, advance an hour more and see
  the run.
- **One live run.** Leave a run unfinished, advance past several moments, and see nothing minted until it settles.

That holds only if the cadence keeps time through the hub's injected clock and nothing else (`bzh:injected-clock`). The
pass reads `now` from it once and hands that instant to the due rule; the schedule module is always handed an instant
and never reads the time; an anchor's instants come from the run's facts, stamped by the same clock. The sweep's own
sixty-second timer is mechanical and stays on real time, which is why a spec steps the pass rather than waiting for it.

## Hub and runners on different versions

A runner sees nothing of a cadence. It claims a cadence's run as it claims any chunk of its project, so a runner of any
supported release works against a hub carrying every piece. The board ships inside the hub, so it always matches the hub
it is served by.

## What an operator does once

Each piece's revision is applied with `blizzard hub migrate`, as every hub revision is (`bzh:manual-migrations`);
nothing else is owed. An operator whose fleet already starts a routine by hand on a rhythm declares a cadence for it,
and stops starting it by hand, once the flag is on.

## Left for later

A cadence that waits for last week's findings to be dealt with before minting again needs a condition on open findings,
which the plan leaves for later. The release train and the store fold wait for a routine that can carry them, since no
fleet chunk's environment holds the workspace where their procedures live. The garden's delivered plans, which foresaw a
cadence as a field on the routine and a routine with no next-run time, stay frozen as they shipped; this plan is the
intent that replaced them.
