---
epic: release-engineering/notes
refinement: refined
status: in-progress
---

# Plan — `epic:release-engineering/notes`

An operator deciding whether to take this week's release opens its notes and finds the fleet's commit log. In an
ordinary week that is close to three hundred subjects across a few dozen chunks, sorted by Conventional Commit type,
written by agents for the reviewers of their own pull requests: `refactor(runner): split the loop context`,
`test(hub):
pin the peek order`. Somewhere in that list are the four things the operator actually needed to know — a new
verb, a changed default, a fix to the stall they hit on Tuesday, a field their dashboard reads going away next major —
and nothing tells them which four. The one section written for them, **Upgrade notes**, is a placeholder that someone
has to remember to replace by hand after the release has already been published, by reading back through the same log to
reconstruct what the release asks.

The person who knew what the operator would notice was the agent that made the change, at the moment it made it. A week
later, at the cut, that knowledge is gone, and the cut is the worst place to rebuild it: whoever is cutting is
summarising work they did not do, against a calendar that will not wait. Deprecations suffer worse. A deprecation is a
promise with a date on it, and today it lives in a sentence in a commit body or a doc, if anywhere, so when a major
arrives the upgrade guide that gathers its removals would have to be written from memory.

This epic moves the note to where the knowledge is. A change an operator would notice carries a short note of its own,
committed beside the code in the same pull request, reviewed with it, and caught by the gate when it is missing. A
change that breaks something carries its upgrade note the same way, and a deprecation names the release that will remove
it. At the cut, the release's notes are assembled from those notes, upgrade notes first; the commit log stays as a
collapsed appendix for the reader who wants everything, never the source. Because every note is a file with fields as
well as prose, a major's upgrade guide is gathered rather than written: every deprecation its cycle announced, each
beside the removal that carried it out.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **A change note, one file per noticeable change.** A note is a small Markdown file in the repository, written in the
  same change as the code it describes: a kind — added, changed, deprecated, removed, fixed, or security — a few
  sentences in the operator's own vocabulary saying what is now true for them, and, when the change asks something of
  them, an upgrade section saying what. One chunk that makes two changes an operator would notice writes two notes; a
  chunk that changes nothing they would notice writes none. A note is reviewed exactly as the code beside it is, and can
  be corrected by any later change until a release carries it.
- **What counts as noticeable, judged twice.** The harness gains a rule that says when a note is owed: anything an
  operator can see, set, or must do differently — a verb, a flag, a configuration key, a board surface, a default, a fix
  to something they could have hit — and nothing they cannot, such as a refactor, a test, or a change to the harness.
  The review node holds every change to it. Beneath that judgement the gate holds what can be checked mechanically: a
  change that alters a published contract, or that carries the breaking marker, fails without a note, and a change that
  alters a contract without anything an operator would notice says so in a line of its commit message rather than in
  silence.
- **An upgrade note for every break.** A breaking change, marked with `!` as today, cannot land without a note carrying
  its upgrade section, and a note claiming a break cannot land without the marker, so the version rules and the notes
  can never disagree about what a release breaks. When a crossing needs more than a paragraph — an order to upgrade in,
  a failure mode, a way back — the note points into a boundary section of the operator's upgrade guide, written in the
  same change, and the guide stays the one home of those instructions.
- **Deprecations that name their removal.** A deprecation note names the release line that will remove what it
  deprecates — a major from 1.0 on, a later minor before it — and the note that finally removes it names the deprecation
  it carries out. The gate refuses a deprecation without a removal line, and a removal of something no released minor
  deprecated is `epic:release-engineering/compat-floor`'s to refuse.
- **Notes assembled at the cut.** Cutting a release gathers every unreleased note into the release's own record and
  renders the published notes from it: upgrade notes first, checked against every break the compatibility check let
  through so none reaches a release unnamed, then what broke or was removed, what is deprecated and when it goes, every
  capability whose flag the release turns on, and then what was added, changed, fixed, and secured. The rendered notes
  become the release's body when it is published; no one edits a published release by hand to finish it. A candidate,
  its promotion, and a patch on a released line each assemble the same way from the notes their line carries.
- **A major's upgrade guide, gathered.** The notes for a major open with one guide to crossing it: every deprecation
  that named that major, in the order they were announced, each with the upgrade section of the removal that carried it
  out. A major whose cycle left a deprecation neither carried out nor moved to a later major does not cut until one or
  the other is done.
- **The commit log as a fallback.** The generated log of Conventional Commits is kept, appended to every release's notes
  as a collapsed section, so the reader who wants the whole history still has it and a note that was missed still leaves
  a trace.
- **Written into the harness.** The rule for when a note is owed and what it must say lives under `standards/`, the
  procedure for writing a change note, an upgrade note, and a deprecation under `workflows/`, and the gate's check in
  the verifiability matrix, so a worker told to make a change already knows the note is part of it.

## Left for later

Notes about a capability that lands dark wait for the release that turns its flag on; which release that is, and when a
flag is on, is `epic:release-engineering/dark-launch`'s, and this epic only holds the note back until then. When the
weekly candidate is cut, how patches reach a released line, and how a major opens `master` for its removals are
`epic:release-engineering/release-lines`'; this epic gives each of those cuts its notes and nothing more. Notes for
`blizzard-mock`, whose changes no operator sees, are not written. Translating notes, a notes feed, and notes on the
board are left until an operator asks for them.
