# Carry-over

The epic lands on a web workspace that has a kit of thirty files and no Storybook, a `fleet/testing` entry that holds a
few spec helpers but no builders, and specs that build each wire shape by hand. It leaves a workshop that every feature
environment serves, a kit the build holds whole, shared pieces shown in the states the fleet rarely holds, and a way of
working that sends every frontend change through the workshop before it is called done.

## The order of landings

Each piece below lands on its own, leaves `master` green, and is proven by the methods it names. Pieces that do not wait
on each other may land in parallel.

1. **The workshop stands up.** The Storybook packages and the install that admits them under `npm ci`, found in the
   order [toolchain.md](./toolchain.md) §The install gives; `.storybook/`'s three files; the `fleet` project's two
   targets; every `tsconfig`'s stories exclude; the three scripts and their `mise` mirrors; the `web/README.md` section;
   and `tests/test_web_workshop.py`. `KitButton` and `KitPanel` get their stories, as proof that a control and a
   container with projected content both render with the board's own styles, zoneless and compiled ahead of time. Proven
   by `web:typecheck`, `web:unit-test`, `web:bundle-composition`, `web:shell-sweep`, and `blizzard:unit-test`, all
   unchanged by the new tree, and by the two parts opened by hand in `npm run storybook`. If neither a direct Vite
   dependency nor a scoped override holds, the piece stops and asks the product owner before relaxing peer checking.
   That question is the piece's to raise, and it is never answered by default.
2. **The workshop in every feature environment.** Waits for piece 1, and lands immediately after it, so that no part's
   stories are written before the agents writing them can reach the workshop. It has two halves. The workspace half —
   the band's `BZ_STORYBOOK_PORT`, the `storybook` service, and the web window's third pane
   ([workspace.md](./workspace.md)) — is the workspace's own change, made by hand on the workspace's `master`, since the
   workspace is no feature environment's project; it is proven by `winter service up <env> --wait` reporting the
   workshop healthy in a fresh environment and the two stories opening at its port. The context half —
   `web:manual-workshop`'s row and section, its guidance for building, the general rows' note, the visual rule's
   admission, and `tool:service-up`'s account — lands in `blizzard-context`.
3. **The render check, and stories among the sweeps.** Waits for piece 1. `story-render-check.js` with its detector
   proof, the `frontend` job's three steps, `ci-gate.sh`'s step, and the widening of every sweep that tells a spec from
   production code ([gates.md](./gates.md) §Stories among the existing sweeps), each with its detector case.
   `web:story-render` enters the matrix and `commands/web/` describes it. Proven by `web:story-render` over piece 1's
   stories, by a story made to throw locally and seen to fail it, and by `web:structural-gate`.
4. **The kit's controls.** Waits for piece 1. Stories for the parts a person acts on: `KitCheckbox`, `KitChips` and
   `KitChip`, `KitOption`, `KitSelect`, `KitSelectRow`, `KitTextInput`, `KitMenu` and `KitMenuPanel`, `KitMenuItem` with
   `KitMenuItemRadio` and `KitMenuItemSubtitle`, `KitTabs`, `KitTabStrip` and `KitTab`, `KitTooltip`, `KitLinkTile`, and
   `KitBackBar`, each in the states [stories.md](./stories.md) §What a part owes asks of it. Proven by
   `web:story-render` and by `web:manual-workshop` over every part.
5. **The kit's chrome and displays, and the gate that holds the kit.** Waits for piece 4. Stories for the rest of the
   kit: `KitPanelHeader` beside piece 1's `KitPanel`, `KitAsyncState`, `KitSkeleton`, `KitDialog`, `KitConfirmDialog`,
   `KitAccordionSection`, `KitMasterDetail`, `KitFactList`, `KitProseBlock`, `KitMarkdown`, `KitBadge`, `KitCountBadge`,
   `KitBeacon`, `KitAvatar`, `KitGlyph`, `KitPaceBar`, and `KitSlotBar`. In the same change, the kit-stories sweep,
   which can pass only once every part is storied, and `bzh:frontend-kit-stories` in `kit.md`, with the routing row that
   names it. Proven by `web:structural-gate` passing over a whole kit, failing when one stories file is deleted, by
   `web:story-render`, and by `web:manual-workshop` over every new part.
6. **The builders.** Waits for nothing; it may land beside any piece above. `fleet/testing/builders/` with every family
   [builders.md](./builders.md) lists, re-exported from `fleet/testing`, and `builders.spec.ts` beside them, which calls
   every builder, pins that two calls return equal records, and pins that every instant a builder writes is an offset
   from `FIXTURE_NOW`. The spec also brings the builders into a compilation before any story or spec consumes them. No
   existing spec moves in this piece. Proven by `web:unit-test` and `web:structural-gate`.
7. **The chunk detail pieces and the artifact views.** Waits for pieces 3, 5, and 6. Stories for `ChunkTimeline`,
   `ChunkTimelineSelection`, `ChunkFacts`, `ChunkEscalation`, `ChunkDelivery`, `ChunkTokenBreakdown`,
   `ChunkArtifactBody` with its four subcomponents, and `ChunkArtifactsPanel`, in the fleet states
   [stories.md](./stories.md) §The composed pieces names. Proven by `web:story-render` and `web:manual-workshop`.
8. **The chunk page's header and tabs, and the transcript viewer.** Waits for pieces 3, 5, and 6, and may land beside
   piece 7. Stories for `ChunkPageHeader`, `ChunkGeneralTab`, `ChunkNodeHistoryTab`, `ChunkTranscriptsTab`, and
   `TranscriptViewer` with `TranscriptSegmentView`. Proven by `web:story-render` and `web:manual-workshop`.
9. **The shell's chrome and the lens control.** Waits for pieces 3, 5, and 6. Stories for `BoardHeader`, `MobileTabBar`,
   and `MobileTitlebar`; the lens control reshaped into a presentational part with `AppNav` as its container, and its
   stories ([stories.md](./stories.md) §The lens control). Proven by `web:unit-test` over the moved and extended specs,
   `web:shell-sweep` over the control's own sweep spec, `web:manual-board` confirming the lens still switches the
   board's view at desktop and phone widths, `web:story-render`, and `web:manual-workshop`.

The epic is delivered when piece 9 lands. Piece 2's workspace half is not a fleet chunk: it is a change to the workspace
by hand, and it has to land before piece 2's context half claims that the workshop is served.

## Specs already written

No spec moves in a sweep. A spec moves onto the builders when a change next touches it. Pieces 7 through 9 touch the
specs of the pieces they story only where a story exposes something the spec should hold, and they move a spec only when
they touch it. A spec that keeps its hand-built literal until then is not a defect: it builds a whole wire shape by hand
and breaks at compile time when the wire moves, as it already does.

## `epic:ui-toolkit` after it

`epic:ui-toolkit` waits for piece 5, when the kit is whole and held, so its survey's candidates are judged in the
workshop beside the parts they would replace. A part it adopts arrives as a kit part, with its stories, under the same
sweep, and nothing about the workshop changes to receive it.
