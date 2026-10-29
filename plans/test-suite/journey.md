---
epic: test-suite/journey
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/journey`

The journey rehearsal is the closest blizzard comes to an overnight run under test. Five issues are ingested, two
grouped and one reordered; they build, review, and deliver, through a multi-repo land, a review that fails and loops, an
answered question, and an escalation; both daemons are killed with SIGKILL mid-run and restarted; and in the morning the
rehearsal checks that everything merged to the bare `main` with its full history, resumed without takeover, ran nothing
twice, left no environment behind, and that `hub status` tells the truth. That is the promise a rewritten daemon would
most need to keep, and today the rehearsal could not hold one to it: it builds its configuration from the Python's
dataclasses, judges the morning through the Python's own invariant checker, and borrows its helpers from the crash
sweep. This epic ports it to Playwright, and closes a gap in the mock fleet every Playwright test stands on.

## Work areas

The consistency check lands before the port, so the rehearsal is written once against the surface it will keep.

### 1. A consistency check the daemons answer

An architecture change. Today `blizzard dev check-invariants` reads each daemon's configuration to find its database and
opens the store itself, so it works only beside the files, and anything that calls it is reading the store by another
name. The hub and the runner each answer the question themselves instead: an authenticated route runs the invariant
checker inside the daemon and reports every violation by its rule, and `blizzard dev check-invariants` becomes a client
of those routes. An operator gets the same question after a bad night, against a fleet whose disks they cannot reach.
The kill-9 crash sweep keeps running the checker in-process.

### 2. The journey in Playwright

The rehearsal moves into the Playwright package as a test of its own, on a hub and a runner host of its own, since its
subject is the whole fleet through a night. Its Python original is deleted in the same change. It takes the documented
configuration files in place of the dataclasses, and the helpers it borrowed from the crash sweep are rewritten for it
rather than shared, since the sweep stays in Python.

The morning-after judgement moves from `Invariants` to what an operator could see for themselves: the merges on the bare
`main` and their history, each chunk's record through the API, the environments left on the host, and `hub status`.
Where a promise has no outside view yet, the rehearsal gets the surface the initiative's rule asks for, one an operator
or an integration could use as well, before it settles for anything narrower. The deeper check the rehearsal runs today,
that both stores still keep every invariant the design rests on, it keeps by asking the daemons through
`blizzard dev check-invariants`, answered as the first work area leaves it. It stays a local rehearsal, run in a
provisioned feature environment, since it mints a real winter workspace and CI carries no winter source.

### 3. The mock fleet's wire parity in CI

`blizzard-mock` mirrors blizzard's wire without importing it, and `tests/test_wire_parity.py` holds the mirror to the
real thing: the mock hub's models against the committed OpenAPI, its vocabularies against blizzard's own, and
`mock-opencode`'s output through the real parser. It needs a sibling `blizzard` checkout, so CI deselects it and it runs
only by hand, which lets a wire change pass both repositories' gates while the mocks go stale beneath every test that
trusts them. The guard joins `blizzard-mock`'s CI, run against `blizzard` `master`. It depends on nothing else in the
initiative and can land whenever it is picked up.

## Out of scope

The kill-9 crash sweep stays in Python, coupled to the implementation on purpose. The gate's speed is
`epic:test-suite/optimization`'s.
