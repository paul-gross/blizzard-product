# Gates

Two checks hold the workshop: one for whether the kit has stories, and one for whether the stories still render. Neither
judges what a story shows. Both sit beside the checks the web workspace already runs, and neither adds a test tier.

## The kit's stories sweep

`web/scripts/structural-gate.js` gains a sweep, `kit-stories`, run in `main` beside the kit floor and proven able to
fail before it is trusted, as every sweep there is (`assertKitStoriesDetectorWorks`).

It reads `fleet/src/lib/kit/index.ts` with the TypeScript compiler and follows each exported name to its declaration.
Every exported class decorated `@Component` or `@Directive` is a **kit part**. It then reads every
`fleet/src/lib/kit/*.stories.ts`, takes each default-exported meta, and collects the identifiers it names as `component`
or lists under `subcomponents`, resolved to their declarations the same way. A kit part named by no meta is a violation,
reported as the part's class and its file:

```text
structural-gate: kit parts with no stories in fleet/lib/kit/:

  KitSlotBar (fleet/src/lib/kit/kit-slot-bar.ts)

Give the part a `<file>.stories.ts` beside it whose meta names it as `component` or under `subcomponents`
([bzh:frontend-kit-stories]); a part that genuinely cannot be shown alone goes in KIT_STORIES_EXEMPT with a one-line reason.
```

`KIT_STORIES_EXEMPT` follows `KIT_FLOOR_EXEMPT_SITES`' idiom: a list of `{ part, reason }` entries, and a doc comment
giving each reason in a sentence. It starts empty, since every kit part today can be shown alone. An entry naming a part
the kit no longer exports, or a part a meta does name, fails the sweep as stale, so the list can only shrink to what it
still excuses. The detector check plants an in-memory kit with one storied component, one storied directive through
`subcomponents`, one unstoried component, and one stale exemption, and fails the gate if the sweep disagrees about any
of them.

The sweep holds the kit and nothing else. A piece outside `fleet/src/lib/kit/` with no stories is never a violation,
wherever it lives ([§The rule](#the-rule)).

## Stories among the existing sweeps

A story is spec-like code: it ships in no bundle, it renders a part rather than serving one, and it may reach test
support. Every sweep that already tells a spec from production code tells a story the same way. Each script's own
`isSpec`, or its inline `.spec.ts` test, widens to `*.spec.ts` or `*.stories.ts`:

| Sweep                                              | What a story is to it                                                                                                                        |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Package layers (`structural-gate.js`)              | May import the project's spec-support unit — `fleet/testing` through the entry, or `testing/` relatively from inside `fleet` — as a spec may |
| Placement (`placement-sweep.js`)                   | Not reach: a hub story importing a `fleet` part does not count as the hub reaching it, as a hub spec does not                                |
| Disjoint diffs (`disjoint-diffs-sweep.js`)         | A consumer, as a spec is                                                                                                                     |
| Wire conformist (`wire-conformist-sweep.js`)       | Out of scope, as a spec is                                                                                                                   |
| Containers compose (`containers-compose-sweep.js`) | Out of scope, as a spec is                                                                                                                   |

The real-timer sweep reads `*.spec.ts` alone and stays so, since a story runs inside no merge gate's clock. Each
widening is added to its sweep's own detector check: a must-pass story that imports `fleet/testing`, and a must-pass hub
story whose `fleet` import does not count as the hub's reach.

`tests/test_web_workshop.py` pins what the scripts cannot see from inside `web/`. The workshop's stylesheets match the
hub build's `styles` ([toolchain.md](./toolchain.md) §What every story renders inside). Every `tsconfig` that compiles
production code or specs excludes `**/*.stories.ts`. And `angular.json`'s `fleet` project carries the `storybook` and
`build-storybook` targets with `zoneless` set.

## The render check

`web/scripts/story-render-check.js` proves that every story in the built workshop renders. It runs over the static build
`npm run storybook:build` leaves in `web/dist/storybook`:

1. Serve the directory on an ephemeral `127.0.0.1` port with a static file server in the script itself, using Node's
   `http` module and adding no dependency.
2. Read the build's `index.json` and keep every entry whose `type` is `story`.
3. Launch headless Chromium through `playwright`, already a `devDependency`, and open one page.
4. For each story, navigate the page to `iframe.html?id=<id>&viewMode=story` and wait until Storybook's preview reports
   the story finished, as `sb-show-main` on the body for a render and `sb-show-errordisplay` for a failure, with a
   ten-second bound.

A story **fails** when the preview shows its error display, when the page raises an uncaught exception, when the page
logs through `console.error`, when a request for one of the workshop's own files fails, or when it does not finish
inside the bound. The script prints each failing story's id and title with the message that failed it, prints a count of
stories rendered, and exits non-zero on any failure. It also exits non-zero when the index holds no story at all, since
an empty workshop passing would be a check that cannot fail.

Before the run is trusted, the check proves it can fail. It serves an in-memory page that raises an exception, another
that logs an error, and a third that never finishes, all from the same server, and refuses to run when it misses any of
them. This is the reasoning behind every `assert…DetectorWorks` in `structural-gate.js`.

The check renders each story once, at the default viewport. It does not compare pixels, query the DOM, or run a story's
`play`, because stories assert nothing ([the epic plan](../index.md)). It stays out of the unit-test builder and out of
`npm run test`.

**Where it runs.** `npm run story-render` builds the workshop and runs the check in one command
([toolchain.md](./toolchain.md) §The commands). In CI, `.github/workflows/gate.yml`'s `frontend` job gains three steps
after the structural gate: install Chromium for Playwright (`npx playwright install --with-deps chromium`), build the
workshop, and run the check. A story that breaks therefore fails the pull request that broke it, through the same
required check the rest of the frontend gate is. `scripts/ci-gate.sh`, the local reproduction behind `mise run gate`,
runs `npm run story-render` as a step after bundle composition, under the method's name as its other web steps are.
`web:shell-sweep` stays a local method: the render check needs Chromium in CI, but taking on a second real-browser suite
is that method's own decision, not this epic's.

## The rule

`blizzard-context`'s `architecture/frontend-structure/kit.md` gains a second rule beside the kit floor, in the slot
skeleton the file already uses:

- **`bzh:frontend-kit-stories` — a kit part arrives with its stories.** *Rule.* Every component or directive the kit
  exports has stories beside it, showing each meaningful state at the widths it answers to; a presentational piece in
  `fleet` outside the kit has them too, as does one in an app's shared shell, such as `hub/src/app/shell/`. *Why.* The
  workshop is only a whole picture of the kit while every part is in it, and a part nobody can see alone is judged
  through whichever page shows it. *Scope.* The kit is held by the build. A presentational piece in `fleet`, or in an
  app's shell, is held by review, and the expectation reaches no further: a component in one feature's own folder, such
  as `hub/src/app/board/` or `runner/src/app/machine/`, may have stories, but nothing asks for them. *Detect.*
  `web:structural-gate`'s kit-stories sweep, with its reasoned exemption list, and `web:story-render` for a story that
  has stopped rendering. *Do.* A new kit part lands with its `.stories.ts` and the states its inputs select, built per
  [stories.md](./stories.md)'s conventions, which the rule restates as its own. *Don't.* A story that wraps a container
  in fake providers so it will render.

The frontend-structure hub's routing row for `kit.md` gains "or adding a part to the kit, or stories to a shared piece".

`verification/blizzard.md` gains two rows. In Commands:

| Method             | Command                                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------------------ |
| `web:story-render` | `npm run story-render` in `web/` — builds the workshop and renders every story in headless Chromium *(more)* |

In Manual testing, `web:manual-workshop` ([workspace.md](./workspace.md) §The manual method). `commands/web/` describes
`web:story-render` in a section of its own — what it runs, that a story's appearance is outside what it proves, and that
it cannot fail on an empty index — and its existing `web:structural-gate` section names the kit-stories sweep and the
story widening among the sweeps. `standards/frontend.md`'s `bzh:frontend-toolchain` keeps its single test runner in the
unit tier: the render check is a build check over stories, not a second runner.
