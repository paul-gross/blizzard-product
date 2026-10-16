# Upgrade notes, deprecations, and a major's guide

## The upgrade section

A note's `## Upgrade` section ([fragments.md](./fragments.md) §The body) is the release's upgrade note for that change.
It says, in the operator's terms, what they must do to take the release: what to upgrade first, what to set or remove,
what they will see if they skip it. It is required on every breaking note and every removal, and allowed on any note
whose release asks something of the operator without breaking anything — a new key worth setting before the first start,
a slow migration to plan around.

The upgrade section is where a change's deployment steps travel. A step an issue's `## Landing and deployment` section
would otherwise ask of whoever deploys the change — a key to set, a halves-in-order restart, a command to run before the
first start — is an operator-facing step, so the change that needs it writes it as its note's upgrade section, and every
operator, blizzard's own instance among them, reads it in the release's **Upgrade notes**. The instance's runbook reads
nothing else at release time (`epic:release-engineering/first-release`'s
[runner-pin.md](../../first-release/spec/runner-pin.md)).

A breaking note also lists in `breaks` the `where` of each break `epic:release-engineering/compat-floor` records for the
change, so the release's `compat-breaks.json` and its upgrade notes are checked against each other at the cut
([assembly.md](./assembly.md) §Breaks cross-checked): no break the static diff found reaches a release without an
upgrade note naming it.

## Boundaries

`docs/upgrade.md` already holds a section per crossing that needs more than a paragraph — an upgrade order, a failure
mode, a way back — such as its runner identity and host graduation boundaries. That stays the one home of such
instructions. A change whose crossing needs one adds its boundary section to `docs/upgrade.md` in the same change, and
its note's upgrade section says in a sentence or two what the crossing asks and sets `upgrade-guide` to the section's
anchor; the assembler links it ([assembly.md](./assembly.md)). The note never restates the section.

A boundary section is written before anyone knows which release will carry it. How a boundary carries its version is
`epic:release-engineering/first-release`'s convention for `docs/upgrade.md`; the cut fills the version in for every
boundary a released note's `upgrade-guide` names, in the same commit that moves the note, so the anchor the note
recorded keeps resolving.

## Deprecations

A deprecation is a note of kind `deprecated` whose `removes-in` names the line that will remove what it deprecates. From
1.0 on that is a major (`"2"`); before 1.0, when a break waits for any minor after the one that marked it deprecated
([initiative](../../initiative.md) §The rhythm), it is a minor (`"0.12"`). At the cut, the assembler refuses a
deprecation whose `removes-in` is not later than the release being cut ([assembly.md](./assembly.md) §The cut), and the
rendered notes list every deprecation the release carries with the line that will remove it.

A released deprecation's `removes-in` is never edited in place ([fragments.md](./fragments.md) §Once released). It moves
to a later line only by a new note — a `changed` note with `postpones` naming the deprecation's id and its own
`removes-in` naming the new line — so a moved date is itself announced. A deprecation's current line is the `removes-in`
of the latest released note that postpones it, or its own when none does.

## Removals

The change that removes a deprecated surface writes a `removed` note with `breaking: true`, an upgrade section, and
`retires` naming the deprecation's id. Whether a break is allowed at all is `epic:release-engineering/compat-floor`'s
([gates.md](../../compat-floor/spec/gates.md) §What lets a break through): a removal a released minor deprecated, or, on
a `master` open for it, a break with no prior deprecation that is declared — a `!` commit and a breaking note naming it
in `breaks`. The note is what carries either to the operator.

## A major's guide

`blizzard-change-notes guide <major>` renders the upgrade guide for crossing into a major from the released notes alone.
It gathers every deprecation whose current line is that major, in the order of the releases that carried them, and under
each its retiring removal's body and upgrade section, linking its boundary when it names one. Every breaking note that
retires nothing — a declared break with no prior deprecation, admitted by compat-floor on an open `master` only with its
`!` and a breaking note naming it ([gates.md](../../compat-floor/spec/gates.md) §What lets a break through) — is listed
after them.

The guide runs as part of a major's cut ([assembly.md](./assembly.md) §The cut) and opens its rendered notes. It refuses
when any deprecation whose current line is the major has no retiring removal in `changes/unreleased/` or a released
directory of that major, naming each; the cut does not proceed until the removal lands or the deprecation is postponed.
Before 1.0 the same gathering runs for any minor that some deprecation names, so the habit holds before the promise
does.
