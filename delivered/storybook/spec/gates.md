# Gates

Two checks hold the workshop: one for whether the kit has stories, and one for whether every story still compiles and
renders. Neither judges what a story shows. Both sit beside the checks the web workspace already runs, and neither adds
a test tier.

## The kit's stories sweep

`web/scripts/structural-gate.js` gains a sweep, `kit-stories`, run in `main` beside the kit floor and proven able to
fail before it is trusted, as every sweep there is (`assertKitStoriesDetectorWorks`).

It reads `fleet/src/lib/kit/index.ts` with the TypeScript compiler and follows each exported name to its declaration.
Every exported class decorated `@Component` or `@Directive` is a **kit part**. It then reads every
`fleet/src/lib/kit/*.stories.ts`, takes each default-exported meta, and resolves the identifier it names as `component`
to its declaration the same way. A kit part no meta names is a violation, reported as the part's class and its file:

```text
structural-gate: kit parts with no stories in fleet/lib/kit/:

  KitSlotBar (fleet/src/lib/kit/kit-slot-bar.ts)

Give the part a stories file beside it whose meta names it as `component` ([bzh:frontend-kit-stories]);
a part that genuinely cannot be shown alone goes in KIT_STORIES_EXEMPT with its reason.
```

`KIT_STORIES_EXEMPT` follows `KIT_FLOOR_EXEMPT_SITES`' idiom: a list of `{ file, class }` entries, and a doc comment
above it giving each entry's reason in a bullet. It starts empty, since every kit part today can be shown alone. An
entry naming a class the kit no longer exports, or a class a meta does name, fails the sweep as stale, so the list can
only shrink to what it still excuses. The detector check plants an in-memory kit with one storied component, one storied
directive, one unstoried component, and one stale exemption, and fails the gate if the sweep disagrees about any of
them.

The sweep holds the kit and nothing else. A piece outside `fleet/src/lib/kit/` with no stories is never a violation,
wherever it lives ([§The rule](#the-rule)).

## Stories among the existing sweeps

A story is spec-like code: it ships in no bundle, it renders a part rather than serving one, and it may reach test
support. Every sweep that tells a spec from production code tells a story the same way, its own `isSpec` or inline
`.spec.ts` test widening to `*.spec.ts` or `*.stories.ts`:

| Sweep                                              | What a story is to it                                                                                                                        |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Package layers (`structural-gate.js`)              | May import the project's spec-support unit — `fleet/testing` through the entry, or `testing/` relatively from inside `fleet` — as a spec may |
| Placement (`placement-sweep.js`)                   | Not reach: a hub story importing a `fleet` part does not count as the hub reaching it, as a hub spec does not                                |
| Wire conformist (`wire-conformist-sweep.js`)       | Out of scope, as a spec is                                                                                                                   |
| Containers compose (`containers-compose-sweep.js`) | Out of scope, as a spec is                                                                                                                   |

Two sweeps stay as they are. The disjoint-diffs sweep tells no spec from production code and already counts every file
as a consumer, a story among them. The real-timer sweep reads `*.spec.ts` alone, since a story runs inside no merge
gate's clock. Each widening is added to its sweep's own detector check: a must-pass story that imports `fleet/testing`,
and a must-pass hub story whose `fleet` import does not count as the hub's reach. `placement-sweep.js`'s comment on
`PLACEMENT_EXEMPT_UNITS`, which says only specs reach `testing`, names stories beside them.

The rules the sweeps enforce say the same. In `blizzard-context`'s `architecture/frontend-structure/placement.md`,
`bzh:frontend-package-layers`' sentence that a `*.spec.ts` in any unit may also import `testing` names a `*.stories.ts`
beside it, and `bzh:frontend-placement`'s reach by "the non-spec code of both `hub` and `runner`" becomes reach by their
code other than specs and stories.

`tests/test_web_workshop.py` pins what the scripts cannot see from inside `web/`. The workshop's stylesheets match the
hub build's `styles` ([toolchain.md](./toolchain.md) §What every story renders inside). Every `tsconfig` that compiles
production code excludes `**/*.stories.ts`. And `angular.json`'s `fleet` project carries the `storybook` and
`build-storybook` targets with `zoneless` set, and with `enableProdMode` off on the build.

## The render check

`npm run story-render` proves that every story compiles and renders, in three steps ([toolchain.md](./toolchain.md) §The
commands).

**It type-checks the stories.** `tsc --noEmit -p projects/fleet/.storybook/tsconfig.json` compiles every story with the
parts and builders it imports, since the workshop's own build checks no types ([toolchain.md](./toolchain.md) §The
build). An arg naming an input the part does not have, or a builder call the wire has moved past, fails here.

**It builds the workshop**, `npm run storybook:build`, in development mode, so Angular reports what it reports only in
development mode: a binding in a story's `render` template to an input the part does not have, which no type check can
see inside a template string, is logged as an error.

**It renders every story.** `web/scripts/story-render-check.js` runs over the static build in `web/dist/storybook`:

1. Serve the directory on an ephemeral `127.0.0.1` port with a static file server in the script itself, using Node's
   `http` module and adding no dependency.
2. Read the build's `index.json`, whose `entries` is an object keyed by story id, and keep every entry whose `type` is
   `story` and whose `subtype` is `story`.
3. Launch headless Chromium through `playwright`, already a `devDependency`, and open one page. An init script attaches
   to the preview's channel, `window.__STORYBOOK_ADDONS_CHANNEL__`, as the preview creates it, and records the first
   verdict the channel announces for the story.
4. For each story, navigate the page to `iframe.html?id=<id>&viewMode=story` and wait, with a ten-second bound, for the
   verdict. `storyRendered`, which the preview emits only after Angular has bootstrapped the story and run its first
   change detection, is a render. `storyErrored`, `storyThrewException`, and `storyMissing` are failures. The body's
   `sb-show-main` class is not a verdict, since the framework sets it before Angular renders anything.

A story **fails** when the channel announces a failure, when the page raises an uncaught exception, when the page logs
through `console.error`, when a request for one of the workshop's own files fails, or when no verdict arrives inside the
bound. Angular reports a template error in a story's first render through `console.error` while still announcing a
render, so the console is watched for every story, not only the channel. The script prints each failing story's id and
title with the message that failed it, prints a count of stories rendered, and exits non-zero on any failure. It also
exits non-zero when the index holds no story at all, since an empty workshop passing would be a check that cannot fail.

Before the run is trusted, the check proves it can fail. From the same server it serves in-memory pages that speak the
preview's channel: one that announces `storyRendered` cleanly and must pass, one that announces `storyErrored`, one that
announces a render and then logs an error, one that raises an exception, and one that never announces anything. It
refuses to run when it misjudges any of them. This is the reasoning behind every `assert…DetectorWorks` in
`structural-gate.js`.

The check renders each story once, at the default viewport. It does not compare pixels, query the DOM, or run a story's
`play`, because stories assert nothing ([the epic plan](../index.md)). It stays out of the unit-test builder and out of
`npm run test`.

**Where it runs.** In CI, `.github/workflows/gate.yml`'s `frontend` job gains two steps after the structural gate:
install Chromium for Playwright (`npx playwright install --with-deps chromium`), then `npm run story-render`. A story
that breaks therefore fails the pull request that broke it, through the same required check the rest of the frontend
gate is, at the cost of a Chromium install on every gate run. `scripts/ci-gate.sh`, the local reproduction behind
`mise run gate`, runs `npm run story-render` as a step after bundle composition, under the method's name as its other
web steps are. `docs/ci.md`'s list of the `web/` frontend checks names the render check, and its local reproduction line
gains `npm run story-render`. `web:shell-sweep` stays a local method: the render check needs Chromium in CI, but taking
on a second real-browser suite is that method's own decision, not this epic's.

## The rule

`blizzard-context`'s `architecture/frontend-structure/kit.md` gains a second rule beside the kit floor, as a section of
its own in the slot skeleton the floor's section uses. Its heading is `bzh:frontend-kit-stories` — a kit part arrives
with its stories — and its slots say:

*Rule.* Every component or directive the kit exports has stories beside it, showing each meaningful state at the widths
it answers to. A presentational piece in `fleet`'s shared areas, `shell/` and `core/`, has them too.

*Why.* The workshop is only a whole picture of the kit while every part is in it, and a part nobody can see alone is
judged through whichever page shows it.

*Scope.* The kit is held by the build. A presentational piece in `fleet`'s `shell/` or `core/` is held by review. The
expectation reaches no further: a piece in one of `fleet`'s feature folders, such as `chunk/`, `transcripts/`, or
`host/`, or in an app of its own, may have stories, but nothing asks for them.

*Detect.* `web:structural-gate`'s kit-stories sweep, with its reasoned exemption list, and `web:story-render` for a
story that has stopped compiling or rendering.

*Do.* A new kit part lands with its stories file and the states its inputs select, built per
[stories.md](./stories.md)'s conventions, which the rule restates as its own.

*Don't.* Wrap a container in fake providers so that it will render.

The frontend-structure hub's routing row for `kit.md` gains "or adding a part to the kit, or stories to a shared piece".

## The matrix

`verification/blizzard.md` gains two rows. In Commands:

| Method             | Command                                                                                                                       |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `web:story-render` | `npm run story-render` in `web/` — type-checks the stories, builds the workshop, and renders every story in headless Chromium |

In Manual testing, `web:manual-workshop` ([workspace.md](./workspace.md) §The manual method).

`verification/blizzard/commands/web.md` splits its methods on whether they need a browser, so `web:story-render` gets a
spoke of its own, `commands/web/story-render.md`, and a routing row in the hub beside `shell-sweep.md`'s. Its section
says what the method runs, that a story's appearance is outside what it proves, and that it cannot pass on an empty
index. `commands/web/static-checks.md`'s `web:structural-gate` section names the kit-stories sweep and the story
widening among the sweeps. `standards/frontend.md`'s `bzh:frontend-toolchain` keeps its single test runner in the unit
tier: the render check is a build check over stories, not a second runner.
