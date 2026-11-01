# OpenCode version lines specification

OpenCode 1 and OpenCode 2 are two **lines** of one harness family. They share the harness id `opencode` and nothing
else: each line has its own binding, admitted range, fixture corpus, runner configuration section, and conversation
store. A line is named by its major version.

## Binding dispatch

The runner's harness registry holds one adapter per configured line, keyed by `(harness_id, line)`. The `opencode` id
resolves to a line in two places only. At capability probe, each configured line's binary reports its version, and the
version must fall inside that line's admitted range; at session mint, selection picks a line, and the session records
it. Every later operation on the session — resume, judgement, answer, nudge, context read, transcript read, takeover —
dispatches through the recorded line's adapter without consulting the binary's current version to choose a binding.

The line is derived from the session's recorded mint-time harness version, which the
[harness selection contract](../../harness-selection.md) already persists. No column is added. A session reference
remains `(harness_id, session_id)` for display and uniqueness; dispatch reads the line beside it. A session whose
recorded line has no available adapter on its runner is unavailable, exactly as a session whose harness is missing, and
the ordinary retry policy mints a fresh session or escalates. Neither line ever resumes a session the other minted.

## Admission

| Line | Admitted range   | Corpus                          |
| ---- | ---------------- | ------------------------------- |
| 1    | `>=1.18.25,<2.0` | 1.18.25, 1.18.32, 1.18.33       |
| 2    | `>=2.1.4,<2.2`   | 2.1.4, written by the 2.x proof |

The 2.x range admits one minor version at a time because the 2.x server API is young and has renamed payload fields
within a major. Moving it is a compatibility proof and a committed corpus inside the new range. The 1.x range is not
changed by this epic.

The runner re-reads the binary's version at every server launch, not only at daemon start. OpenCode 2 ships an `upgrade`
command, so the binary can change beneath a running runner. A launch whose observed version differs from the version the
capability advertised is refused before the server starts, marks the line unavailable, and triggers a fresh capability
probe; a version inside the admitted range becomes available again only through that probe. The 2.x worker configuration
disables OpenCode's own automatic update.

## Runner configuration

The existing `[opencode]` section configures line 1 and keeps its meaning. A new `[opencode_2]` section configures line
2: the binary path, the store directory, model and effort aliases, the operator bundle, and the compaction mapping. A
runner may carry either section, both, or neither. Each binary is named by an absolute path, because both lines install
an executable called `opencode` and `PATH` cannot tell them apart.

A section whose binary reports a version from the other line is a configuration error, not a quiet promotion: the
capability is unavailable and the runner's health names the line the binary actually belongs to.

## Store isolation

A line-2 store is a runner-owned OpenCode data and state directory, defaulting under the runner's own state root. The
runner refuses a line-2 store that is the operator's personal OpenCode data directory, and refuses one that holds a 1.x
store, because OpenCode 2 migrates a 1.x store in place on first start. Isolation is by `XDG_DATA_HOME`,
`XDG_STATE_HOME`, and `XDG_CACHE_HOME` on every line-2 process the runner launches; `HOME` and `XDG_CONFIG_HOME` remain
the operator's, so their skills and user configuration stay discoverable exactly as they are for 1.x, subject to what
the proof records under ambient discovery.

`blizzard runner opencode login --line 2` runs `opencode auth login` against the runner's line-2 store, so the operator
authenticates the store once with their ordinary flow. Capability health treats a store with no usable provider
credential for the configured tiers as unavailable.

## Selection

An acceptable-set entry is a harness id with an optional line selector: `opencode` accepts any available line,
`opencode@2` and `opencode@1` accept only that line. The selector is additive to the acceptable-set grammar the
[harness selection contract](../../harness-selection.md) defines, and every surface that authors or validates a harness
list accepts it. An entry for a harness with no lines, such as `claude_code@1`, is invalid at graph mint.

A bare `opencode` entry on a runner holding both lines resolves to the newest available line whose binding can resolve
the session's authored tiers; the runner may override that preference with `[opencode_2] preferred = false`. Line
preference applies only at mint. Acquisition eligibility counts a line-selected entry as satisfiable only by a runner
advertising that line, so a chunk that needs `opencode@2` waits for a runner that has it.

A runner advertises one capability per available line, each carrying the harness id, the line, and the observed version.
A runner from before this epic advertises no line; the hub reads its `opencode` capability as line 1, which is the only
line such a runner can hold, and so never offers it a chunk that needs `opencode@2`. A runner from before this epic that
is nonetheless handed a line-selected entry reads it as a harness id it does not hold and refuses the mint, so a
line-selected session never runs on the wrong line.
