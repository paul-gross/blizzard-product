# Gates

## Where the check runs

`blizzard-compat` is a static diff that takes seconds, so it runs wherever a change can arrive, and a break is found on
the change that made it.

| Where                                | Job                                                  | Compared against                                           |
| ------------------------------------ | ---------------------------------------------------- | ---------------------------------------------------------- |
| Pull request to `master`             | `gate / compatibility against supported releases`    | the supported releases of the PR's head tree               |
| Pull request to `release/X.Y`        | the same job                                         | every release already cut on `X.Y`                         |
| Push to `master` or a release branch | the same job, in the gate `push.yml` calls           | as for a pull request to that branch                       |
| A `v*` tag                           | `release / compatibility against supported releases` | the supported releases of the tag's tree, excluding itself |

The job lives in `.github/workflows/gate.yml` and runs on every trigger, no longer only on `pull_request`; it checks out
with `fetch-depth: 0` so every tag is present. `release.yml` runs it again before any build, as a backstop for a landing
that reached `master` without the pull-request gate. A run that resolves no supported release passes and says so.

## What lets a break through

A finding fails the job unless every condition in its row holds.

| Where                | A finding passes when…                                                                                                                                                                                                                                                                                                                                                                                                             |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `master`, before 1.0 | it is `deprecated` in every supported release of the change's own tree — which, where the current `floor` line does not deprecate it, means raising `floor` in `src/blizzard/wire/window.toml` in the same change to a line that does — and a commit the change lands carries `!`                                                                                                                                                  |
| `master`, after 1.0  | `src/blizzard/wire/window.toml` carries `opening` — `master` is open for a major — and either the finding passes as before 1.0, or it is declared: a commit the change lands carries `!` and the change carries a breaking change note naming it                                                                                                                                                                                   |
| A release branch     | never — a patch adds or it is not a patch                                                                                                                                                                                                                                                                                                                                                                                          |
| A tag                | its version's major is above the major of every shipped release among the tags — a candidate or the release of a major not yet shipped — and the finding passes as it would on an open `master`; on any other tag it passes as it would on the branch the tag is cut from, so a tag on a release branch admits nothing. The tag's `opening` is never consulted, since the major's last candidate is cut after `opening` is removed |

The check reads the change's `!` from its commits as `_is_acknowledged` does today
(`src/blizzard/tools/wire_compat.py:494`): the pull request's commits on a pull request, the pushed range on a push. The
breaking change note, its form, and the upgrade note a `!` change owes are `epic:release-engineering/notes`', and its
check fails a `!` change that carries none; this check reads only whether the change declares a note naming the finding.
A deprecated-then-removed break needs no more than its deprecation and the `!`.

Raising `floor` is how the window says out loud which runners a change leaves behind, so the break and the window's move
are one reviewed change and never two. Raising it moves the supported releases too: the job computes them from the
change's own tree, so a release below the new floor is no longer a baseline, and only the releases the change keeps
serving must still accept it. A floor that is lowered, or raised without a finding that needs it, fails the job as a
finding of its own, since either is a window moved for no break — save in the change that opens `master` for a major,
where `epic:release-engineering/release-lines`' procedure sets `floor` to the outgoing major's final minor beside
`opening`.

At a tag, the tool writes every finding it let through to `compat-breaks.json` beside the build, one entry per finding
with its surface, `where`, and the release it breaks, and the release workflow attaches it to the GitHub Release.
`epic:release-engineering/notes` renders it into the release's upgrade guide, so the breaks a release carries are named
in its notes by construction.

## The old modes retired

`blizzard-wire-compat` and both its baselines go: `--baseline merge-base` (`resolve_merge_base_baseline`),
`--baseline
deployed` with its `gh run list` resolution (`resolve_deployed_baseline`, `_listed_baseline`,
`_rest_baseline`), and the first-parent walk that attributes a surviving break to the landing that must acknowledge it
(`check_history`, `_first_parent_steps`). A break is now attributed to the pull request that makes it by the pull
request's own job, and a break that bypassed it is caught at the next push and again at the tag. `push.yml`'s
`wire-compat-deployed` job is removed, and `dev-image` needs only `[gate, upper-tiers, dev-build]`, so `edge` advances
on proven commits alone. `mise run wire-compat` becomes `mise run compat`; `tests/test_wire_compat.py`'s history and
baseline cases go, and its classification cases move to `tests/test_compat_wire.py`. `tests/test_push_workflow.py` pins
the new `dev-image` needs.

An unacknowledged break that reaches `master` is repaired as today: a following commit reverts it, since the job
compares the net tree with the releases and not each step.

## The procedure

`blizzard-context`'s `workflows/judging-a-break.md` is the procedure an agent follows when told to judge whether a
change breaks a supported release. It runs `blizzard-compat --json` on the change, and for each finding decides, in
order: whether the finding is real or an artifact of a document the change forgot to regenerate; whether the surface can
be kept beside its replacement, which is always preferred and makes the finding additive; and, when it cannot, whether
the window allows the break now — `deprecated` everywhere and `floor` raisable, or, after 1.0 on an open `master`,
declared with `!` and a breaking change note — or the change must instead deprecate the surface and leave the removal to
a later change. It ends in one of three verdicts, each with what the change must carry: additive, an allowed break with
the `floor` it raises or the note it declares, and the `!` it needs, or a break that must wait, with the deprecation
that starts its clock. The rules it applies live in `bzh:released-surface-additive` and `bzh:store-expand-contract`, and
the procedure restates neither.
