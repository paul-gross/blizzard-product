# A routine's slug and name

A routine carries one field, `name`, and asks it to do two jobs: it is the routine's lineage, the handle every finding,
finding set, proposal, and run records and every chore will link by, so it is fixed at create and never reused; and it
is what the board shows a person. The second job is held hostage by the first: a routine called `arch-sweep` can never
be shown as *Architecture sweep*. Every other record an operator types the handle of — a project, a forge, a work
source, a repository — already keeps the two apart, a `slug` to type and a `name` to read, and this epic brings the
routine into line before cadences start linking to it.

## The two fields

- **`slug`** — the routine's lineage and its handle: lowercase letters, digits, and hyphens, validated as a scope's slug
  is, unique in its project with retired routines included, fixed at create, and never reused. Everything that today
  keys a routine by its name keys it by its slug: the CLI's routine argument, the config document's entry, the lineage
  recorded on findings, finding sets, proposals, and runs, the delta baseline, the trend series, and a chore's
  `routine`.
- **`name`** — the display name: free text, editable at any time like a project's, never a handle, never resolved, never
  recorded on anything the routine produces. A routine created without one is named its slug.

## The transition

The slug is the name as it stands. One additive revision gives every routine a `slug` column filled with its current
`name`, and the project's uniqueness moves to the slug. Because the name was already fixed at create and unique in its
project, every existing name is already a valid lineage, and every row that recorded a routine's name — a finding, a
finding set, a proposal, a work item, a run's identity — already holds that routine's slug. No such row is rewritten;
those columns keep their names and now read as the slug. A name that is not a valid slug — upper case, spaces — keeps it
anyway as a grandfathered slug, since a lineage cannot be rewritten; only a new routine's slug is validated.

## The released surfaces

Every surface keeps answering what it answers today, and gains the slug beside it (`bzh:released-surface-additive`):

- **Views** gain `slug`. `name` stays, and equals the slug until someone edits it.
- **Create** accepts `slug` and `name`. A request with only `name`, as every released client sends, is a routine whose
  slug and name are both that value, exactly as today.
- **Edit** accepts `name`, which until now it refused as immutable. `slug` stays refused.
- **The config document** keys a routine entry by `slug`. An entry with `name` alone, as every document written today
  has, keys by that name as its slug, and so applies as it always has; it says nothing about the display name, so
  re-applying an old document never undoes a name an operator has since edited. An entry stating both keys by `slug` and
  sets `name`. Export writes both.
- **The board** joins by slug everywhere it joined by name: a routine's pages are keyed by its slug, and a routine's
  runs, findings, and baselines are matched to it by slug. It shows the name wherever it shows a routine to a person.
  The board ships inside the hub, so the board and the hub it is served by always move together.
- **The CLI** resolves a routine by slug. A released CLI resolves by name, and keeps working until an operator edits a
  routine's display name, after which it no longer finds that routine by the name it was given. A script reading `name`
  from the API as a key is in the same position. That narrowing of `name`, from handle to display name, is the one
  behavior a released client can see change, and only after an operator uses the new edit; the change note says so.
- **Run titles and charges** show the display name; the charge also names the slug, so a worker can quote the handle.

## Follow-up

An entry keyed by `name` alone, and a create that sends `name` without `slug`, are deprecated, not removed. Their
removal is a standalone `(BREAK)` follow-up, waiting until an operator decides a release may carry it.

## The context half

`blizzard-context`'s `domain/routines-and-scopes.md` says a routine's slug is its lineage, and its name only what it is
called. Blizzard's operator page on routines and scopes and its config-documents page say the same.
