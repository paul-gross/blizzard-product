# Stories

## A story file

Every part has a stories file of its own beside it, named for the part in kebab case: `KitButton` in `kit-button.ts`
gains `kit-button.stories.ts`, and `KitChip`, declared in `kit-chips.ts` beside `KitChips`, gains `kit-chip.stories.ts`.
The file's default-exported meta names the part as its `component`, so the part's inputs reach the controls. A meta
never leans on `subcomponents` to show a second part, since Storybook reads that field only in the docs addon, which the
workshop does not install ([toolchain.md](./toolchain.md) §The framework). A story file is written in Component Story
Format 3: a default-exported `Meta` and one named `StoryObj` export per state.

```ts
import type { Meta, StoryObj } from '@storybook/angular-vite';
import { fn } from 'storybook/test';

import { KitCheckbox } from './kit-checkbox';

const meta: Meta<KitCheckbox> = {
  title: 'Kit/Checkbox',
  component: KitCheckbox,
  args: { label: 'Include stopped chunks', checked: false, disabled: false, checkedChange: fn() },
};
export default meta;

type Story = StoryObj<KitCheckbox>;

export const Unchecked: Story = {};
export const Checked: Story = { args: { checked: true } };
export const Disabled: Story = { args: { disabled: true } };
export const LongLabel: Story = {
  args: { label: 'Include stopped chunks, and every chunk whose graph has since been retired from the project' },
};
```

Every input a reader could sensibly change is an `arg`, so it appears among the controls. Every output is wired to
`fn()`, so the actions panel shows what the part emitted. A part that takes projected content renders through a `render`
template that projects realistic content and binds the part's inputs from `args`. A part that renders only inside
another — `KitTab` in a `KitTabStrip`, `KitMenuItemRadio` in a `KitMenuPanel` — renders through a template that places
it in its parent, and its own inputs are still its args. A story named for a state shows that state and nothing else,
and a story never asserts: no `play` function, no `expect`. Stories render people's views of a part. What the part
promises is held by its specs and by `web:shell-sweep`.

A stories file is held to ESLint's rules and its 400-line cap as the part beside it is.

Titles group the workshop by where a part lives: `Kit/<Part>` for the kit, `Chunk/<Piece>` for `fleet/src/lib/chunk/`,
`Transcripts/<Piece>`, `Shell/<Piece>` for `fleet/src/lib/shell/`, and `Hub shell/<Piece>` for the hub's own chrome. A
part's name in its title is the name a person calls it — `Kit/Select row`, `Chunk/Token breakdown` — never its class
name.

## What a part owes

A part has a story for each of its meaningful states. These are the states that apply to it, chosen by reading its
inputs and its template rather than by checklist:

- **Its ordinary rendering**, with the realistic content the board gives it.
- **Each variant its inputs select** — every variant, size, placement, or mode an input switches between.
- **Disabled**, where the part can be disabled.
- **Empty and loading**, where the part renders data that can be absent or pending: `KitAsyncState`'s three states, a
  list with nothing in it, a skeleton in place of content.
- **Overflowing**, wherever text or items can exceed the part's box: a label long enough to wrap or clip, a list long
  enough to scroll, a number wide enough to push its neighbor.
- **Narrow**, for any part whose layout answers to width — a container query, a wrap, a collapse — rendered at `phone`,
  and at `phone-sm` where the part is reached on the mobile shell ([toolchain.md](./toolchain.md) §What every story
  renders inside).

An overlay shows the states its inputs reach. `KitDialog` and `KitConfirmDialog` take `open` as an input, so their
stories show them open. `KitSelect`'s list, `KitMenu`'s panel, and `KitTooltip` open only on a gesture, so their stories
show the closed trigger in each state, and a reader opens the overlay by hand in the workshop.

A story builds its data from `fleet/testing`'s builders when the part takes a wire shape ([builders.md](./builders.md)),
and from a literal when it takes a primitive or a type of `fleet`'s own. A story holds no state of its own between
renders and reaches no network.

## The kit

Every class `fleet/src/lib/kit/index.ts` exports as a component or directive has a stories file in `fleet/src/lib/kit/`
whose meta names it as `component` ([gates.md](./gates.md) §The kit's stories sweep). These are the parts today:

| Part                  | Declared in                | Stories file                        |
| --------------------- | -------------------------- | ----------------------------------- |
| `KitAccordionSection` | `kit-accordion-section.ts` | `kit-accordion-section.stories.ts`  |
| `KitAsyncState`       | `kit-async-state.ts`       | `kit-async-state.stories.ts`        |
| `KitAvatar`           | `kit-avatar.ts`            | `kit-avatar.stories.ts`             |
| `KitBackBar`          | `kit-back-bar.ts`          | `kit-back-bar.stories.ts`           |
| `KitBadge`            | `kit-badge.ts`             | `kit-badge.stories.ts`              |
| `KitBeacon`           | `kit-beacon.ts`            | `kit-beacon.stories.ts`             |
| `KitButton`           | `kit-button.ts`            | `kit-button.stories.ts`             |
| `KitCheckbox`         | `kit-checkbox.ts`          | `kit-checkbox.stories.ts`           |
| `KitChip`             | `kit-chips.ts`             | `kit-chip.stories.ts`               |
| `KitChips`            | `kit-chips.ts`             | `kit-chips.stories.ts`              |
| `KitConfirmDialog`    | `kit-confirm-dialog.ts`    | `kit-confirm-dialog.stories.ts`     |
| `KitCountBadge`       | `kit-count-badge.ts`       | `kit-count-badge.stories.ts`        |
| `KitDialog`           | `kit-dialog.ts`            | `kit-dialog.stories.ts`             |
| `KitFactList`         | `kit-fact-list.ts`         | `kit-fact-list.stories.ts`          |
| `KitGlyph`            | `kit-glyph.ts`             | `kit-glyph.stories.ts`              |
| `KitLinkTile`         | `kit-link-tile.ts`         | `kit-link-tile.stories.ts`          |
| `KitMarkdown`         | `kit-markdown.ts`          | `kit-markdown.stories.ts`           |
| `KitMasterDetail`     | `kit-master-detail.ts`     | `kit-master-detail.stories.ts`      |
| `KitMenu`             | `kit-menu.ts`              | `kit-menu.stories.ts`               |
| `KitMenuPanel`        | `kit-menu.ts`              | `kit-menu-panel.stories.ts`         |
| `KitMenuItem`         | `kit-menu-item.ts`         | `kit-menu-item.stories.ts`          |
| `KitMenuItemRadio`    | `kit-menu-item.ts`         | `kit-menu-item-radio.stories.ts`    |
| `KitMenuItemSubtitle` | `kit-menu-item.ts`         | `kit-menu-item-subtitle.stories.ts` |
| `KitOption`           | `kit-option.ts`            | `kit-option.stories.ts`             |
| `KitPaceBar`          | `kit-pace-bar.ts`          | `kit-pace-bar.stories.ts`           |
| `KitPanel`            | `kit-panel.ts`             | `kit-panel.stories.ts`              |
| `KitPanelHeader`      | `kit-panel.ts`             | `kit-panel-header.stories.ts`       |
| `KitProseBlock`       | `kit-prose-block.ts`       | `kit-prose-block.stories.ts`        |
| `KitSelect`           | `kit-select.ts`            | `kit-select.stories.ts`             |
| `KitSelectRow`        | `kit-select-row.ts`        | `kit-select-row.stories.ts`         |
| `KitSkeleton`         | `kit-skeleton.ts`          | `kit-skeleton.stories.ts`           |
| `KitSlotBar`          | `kit-slot-bar.ts`          | `kit-slot-bar.stories.ts`           |
| `KitTab`              | `kit-tab.ts`               | `kit-tab.stories.ts`                |
| `KitTabStrip`         | `kit-tab.ts`               | `kit-tab-strip.stories.ts`          |
| `KitTabs`             | `kit-tabs.ts`              | `kit-tabs.stories.ts`               |
| `KitTextInput`        | `kit-text-input.ts`        | `kit-text-input.stories.ts`         |
| `KitTooltip`          | `kit-tooltip.ts`           | `kit-tooltip.stories.ts`            |

A directive or an attribute-selected component — `KitTooltip`, `KitPanelHeader`, `KitMenuItemSubtitle`, `KitTab`,
`KitTabStrip`, `KitLinkTile` — renders through a `render` template that places it on the element it decorates.

## The composed pieces

The pieces `fleet` composes from the kit and shares between the board and the panel get stories once the kit is whole.
Each takes plain inputs; the only thing any of them injects is the clock, which the workshop pins. Their stories files
sit beside them, under `fleet/src/lib/`:

| Area           | Pieces, each with its own stories file                                                                                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Chunk timeline | `chunk/chunk-detail/`: `ChunkTimeline`, `ChunkTimelineSelection`                                                                                                                                        |
| Chunk facts    | `chunk/chunk-detail/`: `ChunkFacts`, `ChunkEscalation`, `ChunkDelivery`, `ChunkTokenBreakdown`                                                                                                          |
| Artifact views | `chunk/chunk-detail/`: `ChunkArtifactBody`, `ChunkArtifactSurvey`, `ChunkArtifactDelta`, `ChunkArtifactBounce`, `ChunkArtifactRawDisclosure`; and `chunk/chunk-artifacts-panel/`: `ChunkArtifactsPanel` |
| Chunk page     | `chunk/chunk-page/`: `ChunkPageHeader`, `ChunkGeneralTab`, `ChunkNodeHistoryTab`; and `transcripts/`: `ChunkTranscriptsTab`                                                                             |
| Transcripts    | `transcripts/`: `TranscriptSegmentView`, and `TranscriptViewer`, which it wraps                                                                                                                         |
| Shell          | `shell/board-header/`: `BoardHeader`; and `shell/mobile-chrome/`: `MobileTabBar`, `MobileTitlebar`                                                                                                      |

Their states are the fleet's, which the dogfood fleet rarely holds at once, and they are what the stories exist to show:

- **The timeline** across a chunk's life: queued with no history, mid-flight on its third epoch, bounced back by a
  review, escalated, landed, and stopped; with and without a selected step; activatable and not; a history long enough
  to scroll.
- **The facts** for a chunk that has every fact and one that has almost none; a graph change pending; neighbors blocking
  and blocked.
- **The escalation** in each form its origin produces: wrapped and raw takeover commands, the raw command alone, prose
  guidance, and nothing to take over.
- **The delivery** before any pull request, with one open, with several repositories landed, and with a landing that
  failed.
- **The token breakdown** for an inexpensive chunk, an expensive one, and one whose usage is still arriving.
- **The artifact body** for each kind and each content an asset carries: a `git_commit`, an asset of plain text, an
  asset whose content is a survey, a delta, or a bounce envelope, and an asset whose content looks structured but fails
  to parse and so renders verbatim; each in `full` and `summary`. **The survey and delta views** alone with one finding
  and with many; **the bounce view** with a cause and detail, with extra keys, and with neither; **the raw disclosure**
  closed and open. **The artifacts panel** empty, with a selection, and holding forty artifacts so its list scrolls.
- **The page header** with no pointers, many pointers, and long blocked-by and blocking lists.
- **Each tab** loading, forbidden, empty, and full, and the general tab with a question awaiting an answer and a
  decision awaiting resolution.
- **The transcript viewer** with a short exchange, a long one, tool calls, and a sidechain; **the segment view**
  truncated, continued from an earlier segment, and continuing in a later one.
- **The board header** with no counts yet, an ordinary day, and a day whose spend outruns its column; **the mobile tab
  bar and titlebar** at `phone-sm` and `phone`, live and not.

Where a piece composes another on the roster — the general tab renders the facts and the timeline, the segment view
renders the transcript viewer — the outer piece's stories show it whole, and the inner piece's stories carry the inner
piece's own states.

## The lens control

The hub shell's project lens control (`web/projects/hub/src/app/shell/nav/project-lens-control.ts`) reads the tenant's
projects through its own `injectProjectsQuery` and switches the lens through an injected `ProjectLens`, so it cannot
render without a live query client and router state. This epic reshapes it into a presentational part, then gives it
stories.

`ProjectLensControl` takes `projects: readonly ProjectView[]` and `current: string | null` as inputs and emits
`choose: string | null`; it keeps `lensChoices` and its sort, and injects nothing. Its container, `AppNav`
(`app-nav.html`), reads the projects through `injectProjectsQuery(() => this.lensEnabled())` and the slug through
`ProjectLens`, binds both, and calls `ProjectLens.switchTo` on `choose` unless the choice is already current. The
control's `enabled` input goes, since the query's enablement now belongs to its container. The derivation stays a plain
function in the control's file, so `AppNav`'s `computed()`s only compose (`bzh:frontend-containers-compose`).
`project-lens-control.spec.ts` moves onto inputs, and `app-nav.spec.ts` gains the wiring. The control's shell-sweep
spec, `project-lens-control.shell-sweep.spec.ts`, keeps mounting `AppNav`, which now owns the query its fixture answers,
so it still proves the whole tab row — route tabs and chips together — across the widths it sweeps.

`project-lens-control.stories.ts`, titled `Hub shell/Project lens`, shows All alone, a handful of projects, a tenant
with enough projects that the chips scroll inside the control, and a long project name, at `desktop`, the only width
`AppNav` renders at. It imports `fleet/testing`'s `projectView` builder through the package entry, as a hub spec would.

## What stays out

A component whose own code injects a query, a mutation, a daemon token, or route state is a container. Its pieces are
storied, but it is not: `ChunkPage`, `ChunkNodeHistoryContainer`, `ChunkTranscriptsContainer`, and every app page.
Components that belong to one app's feature folder stay out as well, and are judged on the page they serve. A story
never wraps a container in fake providers to make it render, because a workshop that simulates the fleet's wiring would
be judged on the simulation.

Presentational pieces in `fleet` that the roster above does not name fall into two groups. Those in `fleet`'s shared
areas — `AppShell` in `shell/`, and `BrandMark`, `FleetWhen`, and `ViewportMenu` in `core/` — gain stories when a change
next touches them, under `bzh:frontend-kit-stories` ([gates.md](./gates.md) §The rule). Those in `fleet`'s feature
folders — `ChunkIssueList`, `ChunkAwaitingHuman`, `ChunkIssuePane`, `ChunkFindingEntry`, `WorkRefLink`, and
`ChunkPageShell` in `chunk/`, and `SubscriptionPaceGroup` and `WorkspaceTable` in `host/` — may gain stories, but no
rule asks for them.
