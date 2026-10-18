# Stories

## A story file

A part's stories live beside it, in one file named for the part's own file: `kit-button.ts` gains
`kit-button.stories.ts`, `chunk-timeline.ts` gains `chunk-timeline.stories.ts`. A file declaring several parts that are
only ever used together — `kit-chips.ts`'s `KitChip` and `KitChips`, `kit-menu.ts`'s `KitMenu` and `KitMenuPanel` — has
one stories file whose meta names the outer part as `component` and the others as `subcomponents`. A story file is
written in Component Story Format 3: a default-exported `Meta` and one named `StoryObj` export per state.

```ts
import type { Meta, StoryObj } from '@storybook/angular-vite';
import { fn } from 'storybook/test';

import { KitButton } from './kit-button';

const meta: Meta<KitButton> = {
  title: 'Kit/Button',
  component: KitButton,
  args: { tone: 'neutral', disabled: false, pressed: fn() },
};
export default meta;

type Story = StoryObj<KitButton>;

export const Neutral: Story = {};
export const Danger: Story = { args: { tone: 'danger' } };
export const Disabled: Story = { args: { disabled: true } };
export const LongLabel: Story = {
  render: (args) => ({ props: args, template: `<fleet-kit-button [tone]="tone">Requeue every chunk on this graph</fleet-kit-button>` }),
};
```

The input and output names above are illustrative; a story binds the part's real ones. Every input a reader could
sensibly change is an `arg`, so it appears among the controls. Every output is wired to `fn()`, so the actions panel
shows what the part emitted. A part that takes projected content renders through a `render` template that projects
realistic content. A story named for a state shows that state and nothing else, and a story never asserts: no `play`
function, no `expect`. Stories render people's views of a part. What the part promises is held by its specs and by
`web:shell-sweep`.

Titles group the workshop by where a part lives: `Kit/<Part>` for the kit, `Chunk/<Piece>` for `fleet/lib/chunk/`,
`Transcripts/<Piece>`, `Shell/<Piece>` for `fleet/lib/shell/`, and `Hub shell/<Piece>` for the hub's own chrome. A
part's name in its title is the name a person calls it — `Kit/Select row`, `Chunk/Token breakdown` — never its class
name.

## What a part owes

A part has a story for each of its meaningful states. These are the states that apply to it, chosen by reading its
inputs and its template rather than by checklist:

- **Its ordinary rendering**, with the realistic content the board gives it.
- **Each variant its inputs select** — every tone, size, placement, or mode an input switches between.
- **Disabled**, where the part can be disabled.
- **Empty and loading**, where the part renders data that can be absent or pending: `KitAsyncState`'s three states, a
  list with nothing in it, a skeleton in place of content.
- **Overflowing**, wherever text or items can exceed the part's box: a label long enough to wrap or clip, a list long
  enough to scroll, a number wide enough to push its neighbor.
- **Narrow**, for any part whose layout answers to width — a container query, a wrap, a collapse — rendered at `phone`,
  and at `phone-sm` where the part is reached on the mobile shell ([toolchain.md](./toolchain.md) §What every story
  renders inside).

An open overlay renders open. `KitDialog`, `KitConfirmDialog`, `KitMenu`'s panel, `KitSelect`'s list, and `KitTooltip`
each have a story whose render template opens them, so the reader sees the open state without a gesture, beside a story
showing the closed trigger.

A story builds its data from `fleet/testing`'s builders when the part takes a wire shape ([builders.md](./builders.md)),
and from a literal when it takes a primitive or a type of `fleet`'s own. A story holds no state of its own between
renders and reaches no network.

## The kit

Every class `fleet/src/lib/kit/index.ts` exports as a component or directive has stories, each named as the `component`
or among the `subcomponents` of a meta in `fleet/src/lib/kit/` ([gates.md](./gates.md) §The kit's stories sweep). These
are the parts today:

| Stories file                       | Names                                                    |
| ---------------------------------- | -------------------------------------------------------- |
| `kit-accordion-section.stories.ts` | `KitAccordionSection`                                    |
| `kit-async-state.stories.ts`       | `KitAsyncState`                                          |
| `kit-avatar.stories.ts`            | `KitAvatar`                                              |
| `kit-back-bar.stories.ts`          | `KitBackBar`                                             |
| `kit-badge.stories.ts`             | `KitBadge`                                               |
| `kit-beacon.stories.ts`            | `KitBeacon`                                              |
| `kit-button.stories.ts`            | `KitButton`                                              |
| `kit-checkbox.stories.ts`          | `KitCheckbox`                                            |
| `kit-chips.stories.ts`             | `KitChips`, `KitChip`                                    |
| `kit-confirm-dialog.stories.ts`    | `KitConfirmDialog`                                       |
| `kit-count-badge.stories.ts`       | `KitCountBadge`                                          |
| `kit-dialog.stories.ts`            | `KitDialog`                                              |
| `kit-fact-list.stories.ts`         | `KitFactList`                                            |
| `kit-glyph.stories.ts`             | `KitGlyph`                                               |
| `kit-link-tile.stories.ts`         | `KitLinkTile`                                            |
| `kit-markdown.stories.ts`          | `KitMarkdown`                                            |
| `kit-master-detail.stories.ts`     | `KitMasterDetail`                                        |
| `kit-menu.stories.ts`              | `KitMenu`, `KitMenuPanel`                                |
| `kit-menu-item.stories.ts`         | `KitMenuItem`, `KitMenuItemRadio`, `KitMenuItemSubtitle` |
| `kit-option.stories.ts`            | `KitOption`                                              |
| `kit-pace-bar.stories.ts`          | `KitPaceBar`                                             |
| `kit-panel.stories.ts`             | `KitPanel`, `KitPanelHeader`                             |
| `kit-prose-block.stories.ts`       | `KitProseBlock`                                          |
| `kit-select.stories.ts`            | `KitSelect`                                              |
| `kit-select-row.stories.ts`        | `KitSelectRow`                                           |
| `kit-skeleton.stories.ts`          | `KitSkeleton`                                            |
| `kit-slot-bar.stories.ts`          | `KitSlotBar`                                             |
| `kit-tab.stories.ts`               | `KitTabStrip`, `KitTab`                                  |
| `kit-tabs.stories.ts`              | `KitTabs`                                                |
| `kit-text-input.stories.ts`        | `KitTextInput`                                           |
| `kit-tooltip.stories.ts`           | `KitTooltip`                                             |

A directive or an attribute-selected component — `KitTooltip`, `KitPanelHeader`, `KitMenuItemSubtitle`, `KitTab`,
`KitTabStrip`, `KitLinkTile` — renders through a `render` template that places it on the element it decorates.

## The composed pieces

The pieces `fleet` composes from the kit and shares between the board and the panel get stories once the kit is whole.
Each takes plain inputs; the only thing any of them injects is the clock, which the workshop pins.

| Area           | Stories file                                                                                                                                                         | Pieces                                                                                                                                                                     |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Chunk timeline | `chunk/chunk-detail/chunk-timeline.stories.ts`, `chunk-timeline-selection.stories.ts`                                                                                | `ChunkTimeline`, `ChunkTimelineSelection`                                                                                                                                  |
| Chunk facts    | `chunk/chunk-detail/chunk-facts.stories.ts`, `chunk-escalation.stories.ts`, `chunk-delivery.stories.ts`, `chunk-token-breakdown.stories.ts`                          | `ChunkFacts`, `ChunkEscalation`, `ChunkDelivery`, `ChunkTokenBreakdown`                                                                                                    |
| Artifact views | `chunk/chunk-detail/chunk-artifact-body.stories.ts`, `chunk/chunk-artifacts-panel/chunk-artifacts-panel.stories.ts`                                                  | `ChunkArtifactBody`, with `ChunkArtifactSurvey`, `ChunkArtifactDelta`, `ChunkArtifactBounce`, and `ChunkArtifactRawDisclosure` as its subcomponents; `ChunkArtifactsPanel` |
| Chunk page     | `chunk/chunk-page/chunk-page-header.stories.ts`, `chunk-general-tab.stories.ts`, `chunk-node-history-tab.stories.ts`; `transcripts/chunk-transcripts-tab.stories.ts` | `ChunkPageHeader`, `ChunkGeneralTab`, `ChunkNodeHistoryTab`, `ChunkTranscriptsTab`                                                                                         |
| Transcripts    | `transcripts/transcript-viewer.stories.ts`                                                                                                                           | `TranscriptViewer`, with `TranscriptSegmentView` as a subcomponent                                                                                                         |
| Shell          | `shell/board-header/board-header.stories.ts`, `shell/mobile-chrome/mobile-tab-bar.stories.ts`, `mobile-titlebar.stories.ts`                                          | `BoardHeader`, `MobileTabBar`, `MobileTitlebar`                                                                                                                            |

Their states are the fleet's, which the dogfood fleet rarely holds at once, and they are what the stories exist to show:

- **The timeline** across a chunk's life: queued with no history, mid-flight on its third epoch, bounced back by a
  review, escalated, landed, and abandoned; with and without a selected step; activatable and not; a history long enough
  to scroll.
- **The facts** for a chunk that has every fact and one that has almost none; a graph change pending; neighbors blocking
  and blocked.
- **The escalation** in each form its origin produces: wrapped and raw takeover commands, the raw command alone, prose
  guidance, and nothing to take over.
- **The delivery** before any pull request, with one open, with several repositories landed, and with a landing that
  failed.
- **The token breakdown** for an inexpensive chunk, an expensive one, and one whose usage is still arriving.
- **Each artifact kind** `ArtifactKind` names, in `full` and `summary` bodies; a survey and a delta with many findings;
  an artifacts panel empty, with a selection, and holding forty artifacts so its list scrolls.
- **The page header** with no pointers, many pointers, and long blocked-by and blocking lists.
- **Each tab** loading, forbidden, empty, and full, and the general tab with a question awaiting an answer and a
  decision awaiting resolution.
- **The transcript viewer** with a short exchange, a long one, tool calls, and a sidechain.
- **The board header** with no counts yet, an ordinary day, and a day whose spend outruns its column; **the mobile tab
  bar and titlebar** at `phone-sm` and `phone`, live and not.

Where a piece composes another on the roster — the general tab renders the facts and the timeline — the outer piece's
stories show it whole, and the inner piece's stories carry the inner piece's own states.

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
`project-lens-control.spec.ts` moves onto inputs, `app-nav.spec.ts` gains the wiring, and
`project-lens-control.shell-sweep.spec.ts` mounts the control with its inputs bound.

`project-lens-control.stories.ts`, titled `Hub shell/Project lens`, shows All alone, a handful of projects, a tenant
with enough projects that the chips scroll inside the control, and a long project name, at `desktop` and `phone`. It
imports `fleet/testing`'s `projectView` builder through the package entry, as a hub spec would.

## What stays out

A component whose own code injects a query, a mutation, a daemon token, or route state is a container. Its pieces are
storied, but it is not: `ChunkPage`, `ChunkNodeHistoryContainer`, `ChunkTranscriptsContainer`, and every app page.
Components that belong to one app's feature folder stay out as well, and are judged on the page they serve. A story
never wraps a container in fake providers to make it render, because a workshop that simulates the fleet's wiring would
be judged on the simulation.

Pieces in `fleet` that the roster above does not name — `ChunkIssueList`, `ChunkAwaitingHuman`, `ChunkIssuePane`,
`ChunkFindingEntry`, `WorkRefLink`, `ChunkPageShell`, `AppShell`, and `host/`'s `SubscriptionPaceGroup` and
`WorkspaceTable` — gain stories when a change next touches them, under `bzh:frontend-kit-stories`
([gates.md](./gates.md) §The rule).
