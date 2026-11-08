# Showing urgency

Everything an operator reads about urgency is decided in the hub and answered on its views: the urgency's own
([record.md](./record.md) §The verbs), the chunk's ([chunk.md](./chunk.md) §In the chunk's views), and a runner's next
claim ([claim-order.md](./claim-order.md) §The walk, explained). The board and the CLI present them and compute nothing
of their own beyond one rule the plan states: a chunk whose urgency is the tenant's default, or whose work is finished,
wears no mark.

## The administration screen

The administration area gains an **Urgencies** tab at `/admin/urgencies`, beside work sources and repositories, built
from the same parts as theirs: the master list and its detail route keyed by the record, the record panel with its facts
and actions, the create and edit dialog, the retired filter, and the change history filtered to the record
(`bzh:frontend-placement`, `bzh:frontend-container-presentational`). Its writes are mutations over the routes, each
settling on refresh (`bzh:frontend-mutation-settles-on-refresh`), and its write controls ask `config:edit`, as the
neighbouring tabs' do.

- **The list** shows the live set in order, most urgent first, each row with its pill as cards will show it, its name
  and slug, and how many open chunks carry it. Rows reorder by drag, as a project's landing order does, each drop one
  position move. The tenant's default is marked on its row.
- **The default** is chosen at the head of the list: a select over the live set that edits the tenant's settings. It is
  set here because it is the one setting that decides which pill counts as ordinary.
- **The form** creates and edits an urgency: the slug, given once at create and shown fixed after; the name; the color,
  through the palette-and-custom picker a project's color uses; and the mark, an emoji typed or an image chosen, either
  one clearing the other. The dialog's field definitions gain a color kind and a mark kind for these.
- **Retiring** opens a dialog. For the default it says the default must move first and offers no confirm. For an urgency
  carrying open work it says how many chunks carry it and asks where they go, the tenant's default chosen and every
  other live urgency offered. Otherwise it confirms. A retired urgency's panel offers enable.

On a phone the screen reads but does not write, and shows the CLI command for each action instead, as every
administration tab does.

## The pill

The pill is a new piece of the board's kit, beside the badge and the swatch, with its own stories (`bzh:frontend-kit`,
`bzh:frontend-kit-stories`): a rounded mark filled with a color given as a palette name or a hex value, lifted for
legibility on dark surfaces as the swatch lifts a custom color, and carrying either an emoji, an image, or a short text,
with an accessible label and tooltip naming the urgency. Its text takes whichever of the board's light or dark ink
contrasts with the fill. The badge stays what it is: a tone, not a color.

The pill shows the urgency's emoji, else its icon, else its name.

## Cards and list rows

The chunk card's view model gains an optional urgency — its name, color, and mark — which the hub board fills for a
chunk that is not finished and whose urgency is not the tenant's default, and leaves empty otherwise. The card draws the
pill at its very top, above the work-reference chips, whenever it is filled. That puts it on the backlog, the ready
queue, the waiting and needs-human lanes, and running work, and keeps it off the done lane, as the plan decides. The
board reads the tenant's urgency set once, beside the chunks, to join each chunk's urgency to its presentation, and both
reads refresh on the events that already invalidate them, together with an urgency's own change.

The list row draws the same pill immediately to the right of the chunk's identifier, under the same rule. The phone's
glance rows, which draw their own markup, carry the mark after the lane icon.

Both lists keep their rank order, on the board and in the list view alike: a pill says a chunk will be reached sooner,
and the next-claim line on the runners page says by whom.

## A chunk's urgency, set by hand

A chunk's detail header shows its urgency beside its status — the pill, or the default's name in plain text — and, for
an operator holding `chunk:control`, opens a menu of the live set in order with the current one checked. The card's
actions menu gains the same choice. Each is the chunk's edit, shown at once from the pending mutation and settled on
refresh, and neither appears on finished work.

Each board form that mints work gains an urgency choice: authoring a hub item and accepting a garden proposal each take
one, the tenant's default chosen; the routine run dialog takes one, *the routine's default* chosen and naming what that
resolves to; a routine's form gains its default urgency, *the tenant's default* among the choices; and a chore's form
gains its override, *the routine's default* among them.

## A runner's next claim

Each runner's row on the runners page gains a **Next** line from its explanation: the chunk it will reach for, linked to
the chunk, with its pill, or *nothing it can claim*. Where the walk passed chunks to get there, the line opens a short
list of them, each with its pill and its reason in words — *blocked on* the prerequisite, with that chunk's status;
*needs a harness or model this runner lacks* — and, where it differs, what a runner that holds would reach for and where
it stops. The line refreshes on the queue's and the chunk's change events, so an operator who raises a prerequisite's
urgency watches the urgent chunk move to the front of the runner's line.

## The CLI

An `urgency` command group, with the conventions of the tenant-wide groups, the urgency resolved by slug:

```text
blizzard hub urgency create SLUG [--name TEXT] [--color COLOR] [--emoji TEXT | --icon FILE] [--after SLUG | --first]
blizzard hub urgency list [--include-retired]       # in order, most urgent first, the default marked
blizzard hub urgency get SLUG                       # its fields, its place, and its open-chunk count
blizzard hub urgency edit SLUG [--name TEXT] [--color COLOR] [--emoji TEXT | --icon FILE] [--clear-mark]
blizzard hub urgency retire SLUG [--move-to SLUG]
blizzard hub urgency enable SLUG
blizzard hub urgency order SLUG...                  # the whole live order
blizzard hub urgency position SLUG (--after SLUG | --first)

blizzard hub settings get
blizzard hub settings edit --default-urgency SLUG
```

`retire` without `--move-to`, on an urgency open work carries, prints the refusal's count and the command to run with
`--move-to`. The existing groups each gain the urgency where they mint or configure work: `chunk set --urgency`,
`chunk ingest --urgency`, `item create --urgency`, `garden-proposal accept --urgency`, `routine run --urgency`,
`routine create` and `routine edit` with `--default-urgency` and `--clear-default-urgency`, and `cadence chore add` and
`chore edit` with `--urgency` and `--clear-urgency`.

The listings say what the board says, in text. `chunk list` and `queue get` gain each chunk's urgency by slug in a
column of its own, and keep listing in the order they do; `queue get` lists in rank order, as the board's ready lane
does. `chunk get` names the urgency on its header line. `runner next RUNNER` prints the runner's explanation: the next
chunk, then each passed chunk with its urgency and reason. The CLI's contract document is regenerated
(`blizzard:cli-contract`).

## Operator documents

Blizzard's operator documentation gains a page on urgencies: rank and urgency as two axes; the set, its order, and the
default; the record's fields with an example; retiring with work moved, and enabling; claim order, with an urgent chunk
behind a prerequisite worked through; hold and pass-over under claim order, including that a runner which holds stops at
an urgent chunk it cannot use; the next-claim line; the CLI; and the board. It is routed from the deployment guide's
index. The neighbouring pages gain what touches them: the config-documents page the `urgencies` and `settings` sections,
their place before routines, and the refusal of a retired urgency; the page on routines and scopes a routine's default
urgency and a run's; the page on cadences a chore's override; and the board's page the tab, the pill, and the next-claim
line.
