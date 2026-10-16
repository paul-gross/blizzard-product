# Supported releases

## The window file

The window is declared once, in `src/blizzard/wire/window.toml`, packaged in the wheel and read at runtime through
`importlib.resources` by the hub ([handshake.md](./handshake.md)) and at check time by `blizzard-compat` from any tree,
HEAD's or a tag's:

```toml
# The oldest line this tree's hub serves, as MAJOR.MINOR.
floor = "0.9"
# The major `master` is open for, as a major number, while it is; absent otherwise.
opening = "2"
```

| Key       | Meaning                                                                                                                                                                                              | Who moves it                                                                                                                             |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `floor`   | The oldest `MAJOR.MINOR` line whose runners this tree's hub accepts, and whose releases its checks hold it to                                                                                        | Before 1.0, a change that removes deprecated surface ([gates.md](./gates.md)); at a major, release-lines' procedure for opening `master` |
| `opening` | The major `master` is open for, as a major number (`"2"`); present from opening `master` for a major until the major's last cut commit, one train before its date. Never present on a release branch | `epic:release-engineering/release-lines`' procedure for opening `master` for a major                                                     |

The window's policy maps onto the file without a second rule. After 1.0, a major's hub serves its own major at any minor
up to its own and the previous major's final minor, so `floor` is that final minor, set when `master` opens for the
major and unchanged until the next. Before 1.0, `0.x` is one major and a break waits for a minor after the one that
deprecated it, so `floor` rises only in the change that removes deprecated surface, to the line that deprecated it. A
hub never serves a runner newer than itself, so the window's other edge is always the hub's own version and is never
written down. `src/blizzard/wire/version_window.py` parses the file into a frozen `VersionWindow(floor, opening)` and
holds the one rule over it, `admits(hub, runner)`, which both the hub and the tool call.

Opening `master` for a major is one change, made by `epic:release-engineering/release-lines`' procedure
([its majors contract](../../release-lines/spec/majors.md)): it adds `opening` and raises `floor` to the outgoing
major's final minor together. The major's candidates, `v(X+1).0.0-rc.K`, are cut from `master` while `opening` is
present, and the commit that cuts its last candidate — the one `release/(X+1).0` branches from, one train before the
date — removes it; the date promotes that candidate and changes nothing in the tree. `opening` is the single signal that
`master` is open for a major. Every check that asks the question — this epic's, and release-lines' own — reads it, and
none infers it from `pyproject.toml`'s version.

A missing file or an unparseable value fails the gate's contract-drift step and refuses the hub's start, naming the
file. An `opening` at or below the major of the newest **stable** release among the tags fails the gate, naming both: a
major already shipped cannot be opened again. The comparison is against what has shipped, never the tree's own version,
since the tree of a major's candidate already carries that major and must still start. The hub reads only `floor` at
runtime and so never judges `opening`.

## The tags

A release is a `v*` tag matching `vMAJOR.MINOR.PATCH` or `vMAJOR.MINOR.PATCH-rc.N`, read from the repository's own tags
(`git tag --list 'v*'`), so every checkout that runs the tool fetches tags and full history (`fetch-depth: 0`). Any
other tag is ignored. A candidate counts as a release: it is published, the dogfood instance runs it first, and a
candidate is promoted to stable unchanged (`epic:release-engineering/release-lines`), so the newest release of a line is
its newest tag, candidate or stable.

`src/blizzard/tools/compat/releases.py` computes the **supported releases** of a tree from its window and the tags:

1. Keep every release at or above `floor`.
2. Take the newest release on the `floor` line, and the newest release of each major at or above `floor`'s major.
3. Collapse duplicates. Before 1.0 this is at most two releases — the newest `0.x` and the newest on the floor line —
   and after it, the newest of the current major and the previous major's final line.

A minor only adds, so the newest release of a line carries every promise its earlier releases made, and holding a change
to it holds the change to them all. A pull request whose base is a release branch (`release/X.Y`,
`epic:release-engineering/release-lines`) is held instead to every release already cut on that line, which begins with
the line's first release.

An empty set — a tree whose `floor` no tag has reached yet — is not an error: the tool reports that no release is
supported and passes, since nothing has shipped that the change could break.

## A release's surface

A release's surface is read from its tag, never rebuilt from its code. Every surface is a committed contract document
held to the code by a drift check ([surfaces.md](./surfaces.md)), so the document a tag's tree holds is exactly what
that release shipped, and the tool reads it with `git show <tag>:<path>`. It never imports, installs, or runs code at a
tag: the classifiers are HEAD's, applied to the tag's documents and HEAD's, so a rule tightened today applies to every
baseline at once.

| Surface         | Document read from the tag                                       |
| --------------- | ---------------------------------------------------------------- |
| hub↔runner wire | `openapi/hub.openapi.json`, through the wire surface's selection |
| HTTP API        | `openapi/hub.openapi.json`, `openapi/runner.openapi.json`        |
| Configuration   | `contracts/config/hub.json`, `contracts/config/host.json`        |
| Trace contract  | `contracts/traces/dictionary.json`                               |
| Egress contract | `contracts/egress/dictionary.json`                               |
| Store schema    | `contracts/store/hub.json`, `contracts/store/runner.json`        |
| The window      | `src/blizzard/wire/window.toml`                                  |

A release that predates a document has no baseline for that surface; the tool says so for that surface and checks the
rest ([carry-over.md](./carry-over.md)).

## The tool

`blizzard-compat` replaces `blizzard-wire-compat`. It lives in `src/blizzard/tools/compat/`: `releases.py` above, one
classifier module per surface under `surfaces/`, `classify_spec_diff` moving there from
`src/blizzard/tools/wire_compat.py` unchanged in what it reports, and `cli.py`.

```bash
uv run blizzard-compat releases --json       # the supported releases themselves
uv run blizzard-compat                       # HEAD against its supported releases
uv run blizzard-compat --line release/0.9    # HEAD against every release on a line
uv run blizzard-compat --against v0.9.2      # HEAD against one named release
uv run blizzard-compat --json                # the findings, machine-readable
```

`blizzard-compat releases` is the one place the supported releases are computed for anything outside this tool. Its
`--json` output is a stable contract other checks read rather than recomputing the set —
`epic:release-engineering/dark-launch`'s `blizzard:flag-lifecycle`, and the fold target of
`epic:release-engineering/release-lines`, whose floor is the oldest entry:

```json
{
  "floor": "0.9",
  "opening": null,
  "releases": [
    { "tag": "v0.9.4", "line": "0.9", "stable": true, "oldest": true },
    { "tag": "v0.11.0-rc.1", "line": "0.11", "stable": false, "oldest": false }
  ]
}
```

`releases` is ordered oldest first, and a field is only ever added to it. With `--line release/X.Y`, it lists every
release cut on that line instead.

Each check run prints the supported releases it resolved and, per surface and per release, either `additive` or each
finding as `Violation(surface, where, reason, deprecated)`, where `deprecated` is whether the baseline marked the
removed or changed surface deprecated. [gates.md](./gates.md) owns which findings fail.
