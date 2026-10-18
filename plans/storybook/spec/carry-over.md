# Carry-over

The epic lands on a web workspace that has a kit of thirty-one files and no Storybook, a `fleet/testing` entry that
holds a few spec helpers but no builders, and specs that build each wire shape by hand. It leaves a workshop that every
feature environment serves, a kit the build holds whole, shared pieces shown in the states the fleet rarely holds, and a
way of working that sends every frontend change through the workshop before it is called done.

## The order of landings

Each piece below lands on its own, leaves `master` green, and is proven by the methods it names. Pieces that do not wait
on each other may land in parallel. A piece that changes `blizzard-context` lands that half there, and proves it by
`blizzard-context:reference-lint`, `blizzard-context:markdown-format`, `blizzard-context:markdown-lint`, and
`blizzard-context:registry-drift`; no half cites a rule or a method that has not landed yet.

1. **The workshop stands up.** The Storybook packages and their peers, and the install that admits them under `npm ci`,
   found in the order [toolchain.md](./toolchain.md) §The install gives; `.storybook/`'s three files; the `fleet`
   project's two targets; the stories exclude in each production `tsconfig`; `npm run storybook` and
   `npm run storybook:build` with their `mise` mirrors; the `web/README.md` section, without the sentence on what holds
   stories; and `tests/test_web_workshop.py`. `KitButton` and `KitPanel` get their stories, as proof that a control and
   a container with projected content both render with the board's own styles, zoneless. Proven by `web:lint`, which
   reaches `.storybook/` and the stories; `web:typecheck`, `web:unit-test`, `web:bundle-composition`,
   `web:structural-gate`, `web:shell-sweep`, and `blizzard:unit-test`, all passing on the new tree;
   `tsc --noEmit -p projects/fleet/.storybook/tsconfig.json` passing over the two stories; and the two parts opened by
   hand in `npm run storybook`. If neither a direct Vite dependency nor a scoped override holds, the piece stops and
   asks the product owner before relaxing peer checking. That question is the piece's to raise, and it is never answered
   by default.
2. **The workshop in every feature environment.** It has two halves. The workspace half — the band's
   `BZ_STORYBOOK_PORT`, the `storybook` service, and the web window's third pane ([workspace.md](./workspace.md)) —
   waits for piece 1 and lands immediately after it, so that no part's stories are written before the agents writing
   them can reach the workshop. It is the workspace's own change, made by hand on the workspace's `master`, since the
   workspace is no feature environment's project. It is proven by `winter service up <env> --wait` reporting the
   workshop healthy in a fresh environment and piece 1's two stories opening at its port. The context half —
   `web:manual-workshop`'s row and section without the building guidance's last sentence, the spoke's preamble, the
   general rows' note, the visual rule's admission, and `tool:service-up`'s account — waits for the workspace half and
   for piece 3, since the method names `web:story-render`.
3. **The render check, and stories among the sweeps.** Waits for piece 1. `npm run story-render` and its `mise` mirror;
   `story-render-check.js` with its detector proof; the `frontend` job's two steps, `ci-gate.sh`'s step, and
   `docs/ci.md`'s two mentions; and the widening of every sweep that tells a spec from production code, each with its
   detector case, and `placement-sweep.js`'s comment ([gates.md](./gates.md) §Stories among the existing sweeps). The
   `web/README.md` section gains that stories are held by the render check. Its context half: `web:story-render`'s
   matrix row, its spoke and routing row, `static-checks.md`'s mention of the widening, and `placement.md`'s two rule
   texts. Proven by `web:story-render` over piece 1's stories, by a story made to throw and another made to bind an
   input that does not exist, each seen to fail it locally, and by `web:structural-gate`, `web:lint`,
   `blizzard:ci-workflows`, `blizzard:gate`, and `blizzard:ci` on the change's own run.
4. **The kit's controls.** Waits for piece 1. A stories file for each part a person acts on: `KitCheckbox`, `KitChips`,
   `KitChip`, `KitOption`, `KitSelect`, `KitSelectRow`, `KitTextInput`, `KitMenu`, `KitMenuPanel`, `KitMenuItem`,
   `KitMenuItemRadio`, `KitMenuItemSubtitle`, `KitTabs`, `KitTabStrip`, `KitTab`, `KitTooltip`, `KitLinkTile`, and
   `KitBackBar`, each in the states [stories.md](./stories.md) §What a part owes asks of it. Proven by
   `web:story-render`, `web:lint`, and `web:manual-workshop` over every part.
5. **The kit's chrome and displays, and the gate that holds the kit.** Waits for pieces 3 and 4. A stories file for each
   remaining kit part: `KitPanelHeader`, `KitAsyncState`, `KitSkeleton`, `KitDialog`, `KitConfirmDialog`,
   `KitAccordionSection`, `KitMasterDetail`, `KitFactList`, `KitProseBlock`, `KitMarkdown`, `KitBadge`, `KitCountBadge`,
   `KitBeacon`, `KitAvatar`, `KitGlyph`, `KitPaceBar`, and `KitSlotBar`. In the same change, the kit-stories sweep,
   which can pass only once every part is storied, and the `web/README.md` section's mention of it. Its context half:
   `bzh:frontend-kit-stories` in `kit.md` with the routing row that names it, `static-checks.md`'s mention of the sweep,
   and the building guidance's last sentence in `web:manual-workshop`, which cites the rule. Proven by
   `web:structural-gate` passing over a whole kit and failing when one stories file is deleted, by `web:story-render`,
   `web:lint`, and by `web:manual-workshop` over every new part.
6. **The builders.** Waits for nothing; it may land beside any piece above. `fleet/testing/builders/` with every family
   [builders.md](./builders.md) lists, re-exported from `fleet/testing`, and `builders.spec.ts` beside them, which calls
   every builder, pins that two calls return equal records, and pins that every instant a builder writes is an offset
   from `FIXTURE_NOW`. The spec also brings the builders into a compilation before any story or spec consumes them. No
   existing spec moves in this piece. Proven by `web:unit-test`, `web:lint`, and `web:structural-gate`.
7. **The chunk detail pieces and the artifact views.** Waits for pieces 3, 5, and 6. A stories file for each of
   `ChunkTimeline`, `ChunkTimelineSelection`, `ChunkFacts`, `ChunkEscalation`, `ChunkDelivery`, `ChunkTokenBreakdown`,
   `ChunkArtifactBody`, `ChunkArtifactSurvey`, `ChunkArtifactDelta`, `ChunkArtifactBounce`,
   `ChunkArtifactRawDisclosure`, and `ChunkArtifactsPanel`, in the fleet states [stories.md](./stories.md) §The composed
   pieces names. Proven by `web:story-render`, `web:lint`, and `web:manual-workshop`.
8. **The chunk page's header and tabs, and the transcripts.** Waits for pieces 3, 5, and 6, and may land beside piece 7.
   A stories file for each of `ChunkPageHeader`, `ChunkGeneralTab`, `ChunkNodeHistoryTab`, `ChunkTranscriptsTab`,
   `TranscriptSegmentView`, and `TranscriptViewer`. Proven by `web:story-render`, `web:lint`, and `web:manual-workshop`.
9. **The shell's chrome and the lens selectors.** Waits for pieces 3, 5, and 6, and for the hub shell change that makes
   the lens selectors presentational ([stories.md](./stories.md) §The lens selectors). A stories file for each of
   `BoardHeader`, `MobileTabBar`, `MobileTitlebar`, `ProjectLensControl`, and `ProjectMenu`. Proven by
   `web:story-render`, `web:lint`, and `web:manual-workshop`.

The epic is delivered when every piece has landed; pieces 7, 8, and 9 may land in any order, and the last of them closes
it.

## Specs already written

No spec moves in a sweep. A spec moves onto the builders when a change next touches it. Pieces 7 through 9 touch the
specs of the pieces they story only where a story exposes something the spec should hold, and they move a spec only when
they touch it. A spec that keeps its hand-built literal until then is not a defect: it builds a whole wire shape by hand
and breaks at compile time when the wire moves, as it already does.
