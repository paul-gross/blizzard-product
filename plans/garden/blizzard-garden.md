# Blizzard's own garden

What this deployment will tend, and what it owes its own corpus before it can. None of this is blizzard's code — it is
the content blizzard brings to the machinery in [machinery.md](./machinery.md), the same way any project brings its own.
A different project brings entirely different beds, routines, and vocabulary, and blizzard never learns what any of them
mean.

## The two beds

Blizzard's garden has two beds. The agent-facing corpus spans four bodies in three repos: the harness rules in
blizzard-context (the biggest accretion surface — every retrospective proposes rules and no standing process removes
anything), the node prompts inside the blizzard repo, the workspace context docs and workflow methodology, and the
skills and agents themselves. And the application code, held against the architectural constraints blizzard-context
declares — layering, dependency direction, seam shape — because the code accretes faster than human review can hold an
architecture line, and a constraint binds only if something checks it.

The beds entangle, and the comment thread is where it shows: the code's density problem is the corpus's gap — the
fleet's verbose priors ratchet density upward, and nothing pushes back until a rule says what a comment may state. Bed
one before bed two: author the rule, then hold the code to it. A template pointing at a standard that does not exist yet
is not a template anyone would write, so the order is not a preference here — it is the only order available. The
`comments` axis is the worked case: its rules are written and its measurement is a command rather than a judgment, which
is where every axis in the table is headed.

## Why the standards live in the harness

Blizzard's strategies point at blizzard-context rather than restating anything, and the reason goes beyond avoiding a
stale copy. A standard in the harness governs at write time as well as at judgment time: the agent writing the code and
the pass judging it read the same sentence, so prevention and enforcement can never drift apart. That is also why no
separate gardening repo exists — a standard homed anywhere but the harness would leave the writing agent blind to it,
and leave the pass pruning downstream what the harness keeps permitting upstream.

`canon:gardening-axes` is winter's convention for organizing this, and blizzard-context's `garden/` section will be the
rule's first instance: one entry per axis, declaring what it evaluates, over which scopes, judged against which
standards — pointing at those standards rather than restating them — and the measurement every run records. The scopes
are a slug list rather than a set of paths, each with a sentence on what it covers, because a scope like `test-files` or
`public-api-surface` has no directory to point at and a run needs the same name to mean the same ground twice. It is the
well-formed thing blizzard's own routines will point their prompts at.

Blizzard declares **one routine per axis, named for the axis it tends**. The platform sees only a routine name and
attributes findings to it; that the name resolves to a registry entry with criteria and a measurement behind it is
winter's convention and blizzard's discipline, invisible to the hub.

## The axes

These are the axes blizzard will declare for itself, spanning both beds. They make the machinery concrete and they
expose which epics each axis's evidence waits on.

| Axis                   | Bed      | Evaluates                                                                                                                                                                                                                                                                                                                               | Judged against                                                                                                                                                                                                                                  | Evidence now → later                                                                                                                 |
| ---------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `comments`             | code     | Comment and docstring weight: retrospective comments, provenance git already owns, restatements of the obvious — keeping only constraints the code cannot show                                                                                                                                                                          | `bzh:comment-locality`, `bzh:comment-encapsulation`, `bzh:prose-budget`, and `bzh:one-prose-home` in `standards/` — mature                                                                                                                      | `mise run prose-check`'s per-root ratchet already measures the weight; LLM judgment carries the locality call                        |
| `tests`                | code     | Test weight: duplicative tests, assert-nothing tests, implementation-welded brittle tests, ceremony tests                                                                                                                                                                                                                               | `standards/tests.md` — absent; authoring it is the axis's first act                                                                                                                                                                             | LLM judgment → mutation results                                                                                                      |
| `architecture`         | code     | Structural drift: layering, dependency direction, seam shape, repository pattern — understated in one line, carried by the whole `architecture/` domain behind it                                                                                                                                                                       | `blizzard-context:/architecture/` — mature                                                                                                                                                                                                      | LLM judgment; checks that prove mechanical graduate into lint or CI — a finding whose check became a gate is the pass's best outcome |
| `agent-facing-context` | guidance | The harness's own shape: files past the size a reader can hold, leaves answering several questions where one hub and its spokes belong, and content filed under a section that does not own it                                                                                                                                          | `canon:auto-load-tax`, `canon:one-question`, `canon:semantic-load`, `canon:by-reader-task`, and `canon:admission-test` — mature                                                                                                                 | `winter lint`'s `file_size` check + LLM judgment for the split and taxonomy calls → analytics read-data                              |
| `context`              | guidance | The guidance's truth, where `agent-facing-context` covers its shape: rules contradicted by what the code now does everywhere, references to files or commands that no longer exist, instructions superseded but never retired                                                                                                           | The code itself and the canon's freshness principles — "the rule says X, the code does Y" is either code drift (an `architecture` finding) or a dead rule (this axis's), and telling which is the judgment that keeps this axis at the LLM rung | LLM judgment against the live codebase                                                                                               |
| `verification`         | code     | The apparatus's reach, where `tests` prunes its excess: can the app run itself against mocks through every edge case — mock-fleet gaps (forge, harnesses, hub/runner, the stubbed fixture scenarios), behaviors with no mock-reachable path (crash paths, fencing races, lease expiry), behaviors whose only proof is the live instance | The verifiability matrix — the canon already names an unprovable behavior as a gap; this axis hunts those gaps on a cadence                                                                                                                     | LLM judgment over matrix vs. behavior inventory → mutation results expose asserted-but-weak coverage                                 |

Every axis declares its measurement and every run records it whether or not it proposes anything — comment-lines and
test-lines per code-line for the weight axes, corpus size and routing depth for `agent-facing-context`, stale-claim and
dead-reference counts for `context`, matrix coverage for `verification` — so the trend is the run's product even when
the findings are none.

## The proposal vocabulary

The platform stores a proposal's class and never interprets it, so the vocabulary is blizzard's to define. Four classes
cover what a response to blizzard's own findings can be:

- **`remediate`** — do the cleanup the findings describe. The campaign: one proposal per theme per area, sized as work
  somebody could actually pick up, with its findings linked rather than pasted into the body.
- **`prevent`** — fix the source so the inflow stops. The missing standard, the exemplar spreading the pattern, the rule
  too fuzzy to follow. Usually the highest-leverage of the three, and the first thing to reach for when a sweep returns
  a great many findings: filing a thousand cleanups while the thing producing them runs untouched is motion, not
  progress.
- **`mechanize`** — retire the judgment itself, moving the check into a lint rule, a config, or a gate so no later run
  pays a model to notice this class again. Carries its case: which finding class it retires, and at what run-cost saved.
- **`escalate`** — hand the problem out of the fleet. Not a cleanup and not a rule, but a statement that what was found
  is past what gardening can absorb and needs a person to plan it elsewhere. Accepting one settles it and mints no work
  item, because there is no item that would honestly describe "somebody sit down and decide what to do about this."

Recording which of the four a proposal is, is what makes the vocabulary worth having. A routine whose accepted proposals
are all remediation is being weeded while its source runs untouched; a routine producing `escalate` month after month is
one whose scope was drawn wrong, and neither is a fact any count of open findings would show.

Three of those classes are rungs on a path a check travels: prose standard → judged finding → lint rule → CI gate.
Ambiguity is what legitimately keeps a check at the judged rung, and there is no shame in leaving one there. But when a
check does graduate it **moves house rather than being copied** — once a class is a lint rule, the strategy that used to
judge it points at the mechanized check and stops re-judging it. One owner per check, at whatever rung it lives; a check
judged in two places is a check whose two owners will disagree.

The success metric follows from that. A garden that shrinks its own LLM surface over time — judgment migrating into
reflexes until the passes spend their context only on what genuinely needs a mind — is the one working best, and the
store answers whether it is happening: a finding class's recurrence across runs, and whether a human ever passed on the
proposals it raised, make "this class recurs and nobody overrides it" a query rather than somebody's memory.

## Two garden states, and what blizzard does in each

A pass judges by whatever its prompt names, and that one fact makes its behavior proportionate to the condition it finds
the garden in.

In a **healthy garden**, silence is the success state and it must be cheap. A converged run delivers zero additions, a
few still-fine confirmations, and a measurement on its run row — the flat trendline is what upheld gardening looks like
as evidence rather than as a feeling.

An **overgrown garden** is a signal before it is a workload, and blizzard's passes are told to treat it that way before
they treat it as anything else. Every survey opens with a gut-check: could this scope be inventoried well within one
agent's context, or would recording what is here exhaust it? When the honest answer is the second, the run stops. It
enumerates nothing and records a single finding — class `excessive-scope`, the scope itself as its locus, the honest
count or estimate in its summary — and that one finding is its entire output.

Short-circuiting beats the alternative it replaces, which is a partial inventory. A truncated list of a thousand
instances mints a few hundred real findings that read as the whole set, and every later run inherits the confusion. One
finding saying this ground is past what a pass can hold is a fact a person can act on, and it converges: the next run's
reconcile matches it against the live one and records that it was seen again, so a weekly routine against overgrown
ground accumulates one finding and a trail of observations rather than a fresh pile every week.

The response is an `escalate` proposal, and accepting it mints no work item — there is no work item that honestly
describes the judgment it asks for. What a person does next is rescope the routine, author the standard the volume is
really reporting, or plan a campaign somewhere blizzard is not. Because a run that finds thousands has almost certainly
met a **shift in grading**: a standard newly authored, or newly sharpened, that the existing code was never written
against. That is a real and useful thing for a pass to surface, and it is not what gardening is for. The response is
upstream, not a thousand cleanups.

Below that threshold the ordinary posture holds. Stop the inflow first with a `prevent` proposal naming the rule or the
exemplar. Say the number honestly, because the count is the finding that matters whether or not every instance behind it
is enumerated. Aggregate into campaigns — a `remediate` proposal per area, sized as an executable chunk of work — so the
inventory stays queryable without any of it reaching the backlog. And propose boundedly, letting the rest converge. A
pass never files a thousand items: the backlog is a human triage surface, and per-instance filing confuses inventory
with work. Draining a backlog that large is a different problem than tending a garden, and if these passes keep exposing
it, it earns its own machinery rather than this one.

Blizzard's own target sits in the middle band: one context holds most of a scope, so whole-scope runs with faceted
fan-out are the default, and delta mode becomes the steady state once each routine has had its first full sweep.

## What blizzard owes before it can garden

- **`standards/tests.md`** — the one axis in the table with nothing behind it. It is written by hand, by the harness
  engineer, before any tests routine exists; until then there is simply no strategy telling anything to run.
- **The `garden/` section in blizzard-context** — the registry above, authored in canon's four-field shape so every
  routine has one place to point at.
- **A scope list per axis** — the slugs a run of that axis may be scoped to, inside those entries. The registry names
  which scopes an axis sweeps; what each one covers is the hub scope entity's sentence, written when the scope is
  created and not restated here. An axis whose scopes are undeclared is one whose buckets drift the first time an agent
  invents a name that reads like an existing one.
- **The proposal vocabulary, written down** — the four classes above belong in the same registry, because a class the
  hub never interprets is only as stable as the list an agent read before choosing one.

## Evidence, in stages

What a blizzard pass can act on arrives over time, and the table's last column tracks it. Available immediately: the
canon's mechanical shape checks (routing-hub integrity, the auto-load tax, oversized files, rules nothing cites), the
mature `architecture/` rules, git history, and whatever retrospective material is locally reachable. Arriving with
`epic:transcripts` and `epic:analytics`: the usage truth — which files are actually read, which skills actually fire.
Arriving with `epic:mutation-testing`: measured test weight — a test that kills no mutants some other test doesn't also
kill is dead weight by evidence, not judgment.
