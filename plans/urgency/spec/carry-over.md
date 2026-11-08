# Carry-over

The epic lands on a hub whose ready queue has one order, which runners reach in and operators arrange by hand. It leaves
a hub where every chunk carries an urgency from a set each tenant keeps, where a runner reaches for the most urgent
group first and rank within it, and where the board says, for each runner, what it will take next and why. Nothing
deployed changes id, a released client reads every response it read before, now with one optional field more, and no
runner needs upgrading.

## Every tenant has `standard`

The first piece's revision gives every tenant that exists the urgency a new tenant starts with. For each tenant, in one
additive revision (`bzh:store-expand-contract`):

- an urgency with slug and name `standard`, color `slate`, no mark, position `0`, `created_by` `migration`, revision 1,
  and its `create` change row from the `migration` door, its id minted deterministically from the tenant and the
  revision's instant, by a rule the revision restates rather than imports (`bzh:frozen-revisions`);
- the tenant's settings, `default_urgency` naming it, revision 1, with its change row;
- every chunk the tenant holds, finished or open, given `standard`, in bounded pages of a thousand rows.

Each step skips what a run stopped part-way already did, so the revision is safe to run again, and its migration test
proves that, along with a tenant of chunks at every status, a tenant with none, and two tenants each getting their own.
From the next instant on, the queue claims exactly as it did the day before, since one urgency gives claim order the
shape of rank order, and keeps doing so until an operator adds a second.

A tenant created afterwards gets the same urgency and settings inside its creation's own unit of work, so no tenant ever
exists without a default. The hub's crash-correctness record of tenant creation names the two writes it gains.

## The order of landings

Each piece lands on its own, leaves `master` green, and is proven by the methods it names. A piece that changes
`blizzard-context` lands that half there, proven by `blizzard-context:reference-lint`,
`blizzard-context:markdown-format`, `blizzard-context:markdown-lint`, and `blizzard-context:registry-drift`, and no half
cites a rule or method that has not landed. Every piece that changes an operator surface regenerates its contract
documents and carries its change note (`blizzard:change-notes`). The one piece that adds a field the fleet wire carries
extends `blizzard-mock`'s mirror in the same change.

1. **The record.** Waits for nothing. The revision above, with the four tables, the `chunks.urgency_id` column, and the
   `urg` id; the urgency package with its model, its validation, its mark, and its verbs, declared in the domain's
   layering; the settings record; the set's order and its two routes; retiring with work moved and enabling; the
   urgency's keyed lock and its place in the lock order; tenant creation's seed; every mint writing the tenant's
   default, the chain's last link, through a mint that now requires an urgency; the `urgencies` and `settings` sections
   of the config document, their schemas, their reconcile before routines, and their export; the `urgency` and
   `settings` CLI groups ([record.md](./record.md), [chunk.md](./chunk.md) §The column, §Setting an urgency). The
   operator page on urgencies, with the set, the default, and retiring, and the config-documents page's sections. Its
   context half: the domain's new urgency spoke, with the record, the default, and retiring, routed from the work hub;
   the configuration spoke's entries for the urgency and the tenant's settings, the urgency's departure from
   `bzh:config-apply` among them; the hub's crash-correctness record of tenant creation. Proven by `blizzard:unit-test`
   over every validation refusal — a slug a retired urgency holds, an emoji carrying a letter, an SVG icon, both marks
   at once — over the order's refusals, and over every retire refusal in its order; `blizzard:component-test` over the
   revision on a store with chunks at every status, run twice, over tenant creation, over a retire moving open work
   while finished work keeps its urgency, over an apply naming a retired urgency refused, an apply whose order leaves an
   urgency out refused, and an export applied back writing nothing; `blizzard:postgres-uow-concurrency` over a retire
   racing a mint and a chunk edit naming the same urgency; `blizzard:manual-postgres-migrations`;
   `blizzard:store-contract`, `blizzard:cli-contract`, `blizzard:compat`, and `web:client-drift`.
2. **The chunk's urgency.** Waits for piece 1. The chain's stated links on ingest, a hub item's create, a proposal's
   accept, and a routine run; the routine's default and the chore's override, with their columns in one additive
   revision; `urgency` on the chunk's edit, its window, and the queue change it publishes; `urgency` on every chunk
   view; the CLI options that set them ([chunk.md](./chunk.md)); a retire clearing the routine defaults and chore
   overrides that name it; and `blizzard-mock`'s mirror of the views' new field. The pages on routines and scopes and on
   cadences gain what touches them. Its context half: the chunk spoke names the urgency among a chunk's
   operator-editable properties, its window, and that grouping keeps the survivor's. Proven by `blizzard:unit-test` over
   the chain at every link — a run stating one, a chore naming one over a routine naming another, a chore naming none
   over a routine naming none — and over the edit's window at every status; `blizzard:component-test` over each mint
   path writing what the chain resolves, a stated retired urgency refused, grouping keeping the survivor's urgency, an
   edit leaving rank untouched, a retire leaving a routine's default and a chore's override reading as none, and a
   document declaring an urgency and a routine naming it applying in one pass; `blizzard:service-test` editing a ready
   chunk's urgency and reading its view and the queue's change frame; `blizzard:store-contract`,
   `blizzard:cli-contract`, `blizzard:compat`, and `web:client-drift`.
3. **Claim order.** Waits for piece 2. Claim order in the queue operations; the matched and legacy peeks walking it; the
   walk returning its trace ([claim-order.md](./claim-order.md) §Two orders over one queue, §The peeks walk claim
   order). The operator page on urgencies gains claim order, hold and pass-over under it, and an urgent chunk behind a
   prerequisite. Its context half: the ranking spoke names claim order beside each list's rank, and the acquisition
   spoke restates the walk over claim order and says a prerequisite is never raised. Proven by `blizzard:unit-test` over
   the walk — pass-over stepping past a blocked urgent chunk to the next urgent one, then down a group; hold stopping at
   an urgent chunk it cannot use, per project when projects are named; one urgency giving rank order exactly; a reorder
   of the set reordering the walk — and over the trace matching the choice on every fixture; `blizzard:component-test`
   over the peeks on a two-project tenant; `blizzard:service-test` with a runner claiming an urgent chunk ranked last;
   `blizzard:compat`; and `blizzard:manual-hub` with a real runner taking the urgent chunk first.
4. **The explanation.** Waits for piece 3. The next-claim route and `runner next` ([claim-order.md](./claim-order.md)
   §The walk, explained, [surfaces.md](./surfaces.md) §The CLI). Proven by `blizzard:unit-test` over each reason and
   over `held_at`; `blizzard:service-test` reading a runner's explanation before and after raising a blocking
   prerequisite's urgency; `blizzard:cli-contract`; `blizzard:compat`; and `web:client-drift`.
5. **The board.** Waits for pieces 1, 2, and 4. The pill in the kit, with its stories; the Urgencies tab; the pill on
   cards, list rows, and glance rows; a chunk's urgency in its header and its actions menu; the urgency choice on each
   form that mints or configures work; the Next line on the runners page ([surfaces.md](./surfaces.md)). The board's
   operator page. Proven by `web:unit-test` over the view model leaving the pill off a default chunk and a finished one,
   `web:lint`, `web:typecheck`, `web:story-render`, `blizzard:ui` over creating an urgency with an emoji, reordering the
   set, marking a ready chunk urgent and reading its pill and the runner's Next line, and retiring the urgency with its
   chunks moved, and `web:shell-sweep` on a narrow screen.

The epic is delivered when piece 5 lands. It deprecates nothing, so it owes no follow-up removal. It rides behind no
flag: until an operator creates a second urgency, every piece leaves the fleet doing exactly what it did, and a set
built from the CLI before the board's piece lands simply shows no pills until it does.

## Hub and runners on different versions

A runner reads nothing of urgency; the optional field its views gain is one it ignores. It peeks as it always has, and
the hub walks claim order on its behalf, so a runner of any supported release works against a hub carrying every piece
and reaches for urgent work first. A released CLI reads the chunk's views and lists them without the urgency column, and
a document it applies, which names no urgency, applies as it did. A hub older than the epic refuses a document carrying
`urgencies` or `settings`, since its document forbids unknown sections. The board ships inside the hub, so it always
matches the hub it is served by.

## What an operator does once

The revision is applied with `blizzard hub migrate`, as every hub revision is (`bzh:manual-migrations`); nothing else is
owed. Walking the store back across it drops the set and every chunk's urgency, and walking forward again gives every
chunk `standard`, so an operator who rolls a hub back re-creates the urgencies they had and re-marks their urgent work.
An operator who wants urgency adds a second urgency, places it, and marks the work that should jump the line.

## Left for later

`epic:tagging` reads urgency rather than holding it: its inferred tag `urgency:<slug>` is derived from the slug a
chunk's urgency carries, which this epic guarantees is fixed at create and never reused, so the tag means the same thing
for the life of the tenant. The tag, and a runner's declaration over it, are that epic's to build. The experiments of
`epic:lab` name an urgency for their trials the way a routine names its default, and `epic:chokepoint/chokepoint` reads
claim order for ahead and behind; each builds on this epic's record and claim order without this epic building anything
for them.
