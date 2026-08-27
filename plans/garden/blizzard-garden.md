# Blizzard's own garden

What this deployment will tend, and what it owes its own corpus before it can. None of this is blizzard's code — it is
the content blizzard brings to the machinery in [index.md](./index.md), the same way any project brings its own. A
different project brings entirely different axes, and blizzard never learns what they mean.

## The two beds

Blizzard's garden has two beds. The agent-facing corpus spans four bodies in three repos: the harness rules in
blizzard-context (the biggest accretion surface — every retrospective proposes rules and no standing process removes
anything), the node prompts inside the blizzard repo, the workspace context docs and workflow methodology, and the
skills and agents themselves. And the application code, held against the architectural constraints blizzard-context
declares — layering, dependency direction, seam shape — because the code accretes faster than human review can hold an
architecture line, and a constraint binds only if something checks it.

The beds entangle, and the first pull of the comment thread showed how: the code's density problem was the corpus's gap
— the fleet's verbose priors ratchet density upward, and nothing pushes back until a rule says what a comment may state.
Bed one before bed two: author the rule, then hold the code to it. A template pointing at a standard that does not exist
yet is not a template anyone would write, so the order is not a preference here — it is the only order available.

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
well-formed thing blizzard's own templates will point their prompts at.

## The axes

These are the axes blizzard will declare for itself, spanning both beds. They make the machinery concrete and they
expose which epics each axis's evidence waits on.

| Axis           | Bed      | Evaluates                                                                                                                                                                                                                                                                                                                               | Judged against                                                                                                                                                                                                                                  | Evidence now → later                                                                                                                 |
| -------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `comments`     | code     | Comment and docstring weight: retrospective comments, provenance git already owns, restatements of the obvious — keeping only constraints the code cannot show                                                                                                                                                                          | `standards/comments.md` — to be authored as this axis's first act, including the call on the exemplar's citation-docstring style                                                                                                                | LLM judgment; weight-ratio trend                                                                                                     |
| `tests`        | code     | Test weight: duplicative tests, assert-nothing tests, implementation-welded brittle tests, ceremony tests                                                                                                                                                                                                                               | `standards/tests.md` — absent; authoring it is the axis's first act                                                                                                                                                                             | LLM judgment → mutation results                                                                                                      |
| `architecture` | code     | Structural drift: layering, dependency direction, seam shape, repository pattern — understated in one line, carried by the whole `architecture/` domain behind it                                                                                                                                                                       | `blizzard-context:/architecture/` — mature                                                                                                                                                                                                      | LLM judgment; checks that prove mechanical graduate into lint or CI — a finding whose check became a gate is the pass's best outcome |
| `corpus`       | guidance | The harness's own shape and weight: routing integrity, auto-load tax, oversized files, uncited rules, consolidation and re-routing candidates                                                                                                                                                                                           | The canon's structure rules — mature                                                                                                                                                                                                            | Canon checks + retrospective mining → analytics read-data                                                                            |
| `context`      | guidance | The guidance's truth, where `corpus` covers its shape: rules contradicted by what the code now does everywhere, references to files or commands that no longer exist, instructions superseded but never retired                                                                                                                         | The code itself and the canon's freshness principles — "the rule says X, the code does Y" is either code drift (an `architecture` finding) or a dead rule (this axis's), and telling which is the judgment that keeps this axis at the LLM rung | LLM judgment against the live codebase                                                                                               |
| `verification` | code     | The apparatus's reach, where `tests` prunes its excess: can the app run itself against mocks through every edge case — mock-fleet gaps (forge, harnesses, hub/runner, the stubbed fixture scenarios), behaviors with no mock-reachable path (crash paths, fencing races, lease expiry), behaviors whose only proof is the live instance | The verifiability matrix — the canon already names an unprovable behavior as a gap; this axis hunts those gaps on a cadence                                                                                                                     | LLM judgment over matrix vs. behavior inventory → mutation results expose asserted-but-weak coverage                                 |

Every axis declares its measurement and every run records it whether or not it proposes anything — comment-lines and
test-lines per code-line for the weight axes, corpus size and routing depth for `corpus`, stale-claim and dead-reference
counts for `context`, matrix coverage for `verification` — so the trend is the run's product even when the findings are
none.

## What blizzard owes before it can garden

- **`standards/tests.md`** — the one axis in the table with nothing behind it. It is written by hand, by the harness
  engineer, before any tests template exists; until then there is simply no row telling anything to run.
- **The `garden/` section in blizzard-context** — the registry above, authored in canon's four-field shape so every
  strategy has one place to point at.
- **A scope list per axis** — the names a run may be scoped to and a sentence each on what they cover, inside those
  entries. Blizzard indexes the name and resolves nothing, so an axis whose scopes are undeclared is one whose buckets
  drift the first time an agent invents a name that reads like an existing one.

## Evidence, in stages

What a blizzard pass can act on arrives over time, and the table's last column tracks it. Available immediately: the
canon's mechanical shape checks (routing-hub integrity, the auto-load tax, oversized files, rules nothing cites), the
mature `architecture/` rules, git history, and whatever retrospective material is locally reachable. Arriving with
`epic:transcripts` and `epic:analytics`: the usage truth — which files are actually read, which skills actually fire.
Arriving with `epic:mutation-testing`: measured test weight — a test that kills no mutants some other test doesn't also
kill is dead weight by evidence, not judgment.
