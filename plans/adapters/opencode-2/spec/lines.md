# OpenCode version lines specification

OpenCode 1 and OpenCode 2 are two **lines** of one harness family. They share the harness id `opencode` and nothing
else: each line has its own binding, admitted versions, fixture corpus, runner configuration section, and conversation
store. A line is named by its major version.

## Binding dispatch

Each configured line is its own adapter, and every structure the runner and hub key by harness id today is keyed by
`(harness_id, line)` for `opencode`. On the runner that is the harness registry and its wiring
(`runner/harness/wiring.py`, `runner/harness/registry.py`), the health cache, the capability snapshot, the
`selftest_results` table, and the `blizzard runner selftest` argument, which accepts `opencode@2`. On the hub it is
eligibility (`hub/domain/execution/eligibility.py`), which today takes the first capability whose id matches and would
judge tiers against one line only; it judges each advertised line separately. The mock hub's matching mirrors the hub.

The line is fixed at mint. A session's line is the line of the harness version recorded on its first identified spawn
for `(harness_id, session_id)`; a session whose first spawn recorded no version is line 1, which covers every session
minted before this epic. A line-2 mint refuses to record a spawn without an observed version, so no line-2 session can
fall into that default. No column is added. Every later operation on the session — resume, judgement, answer, nudge,
context read, transcript read, takeover — dispatches through that line's adapter, without consulting the binary's
current version to choose one.

A session whose line has no available adapter on its runner is unavailable, as a session whose harness is missing is,
and the attempt escalates in place. Neither line ever resumes a session the other minted.

## Admission

| Line | Admitted versions | Corpus                                        |
| ---- | ----------------- | --------------------------------------------- |
| 1    | `>=1.18.25,<2.0`  | `contracts/opencode/1.18.25/`                 |
| 2    | `==2.1.4`         | `contracts/opencode-2/2.1.4/`, from the proof |

Line 2 admits exactly the versions it has proven, as a set of pinned versions rather than a range, because the 2.x
server API is young and has renamed payload fields within a major. Admitting another version is a compatibility proof
run against it and its corpus committed beside the first. Line-2 corpora live under their own `contracts/opencode-2/`
root, so line-1 corpus guards that walk `contracts/opencode/` never see a 2.x capture. Line 1's range is not changed by
this epic.

The runner already reads the binary's version at every launch. What line 2 adds is the comparison: a launch whose
observed version differs from the version the capability advertised is refused before the driver starts a server, marks
the line unavailable, and triggers a fresh capability probe; the line becomes available again only through that probe.
Every line-2 process carries `OPENCODE_DISABLE_AUTOUPDATE=1`.

## Runner configuration

The existing `[opencode]` section configures line 1, and every key in it keeps its meaning, including a `binary` found
on `PATH`. A new `[opencode_2]` section configures line 2:

| Key                  | Meaning                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------- |
| `enabled`            | Whether the line is configured at all                                                         |
| `binary`             | Absolute path to the 2.x executable; required, because both lines install a binary `opencode` |
| `store`              | The line-2 store directory, defaulting under the runner's state root                          |
| `worker_config_path` | The runner-owned worker configuration, as for line 1                                          |
| `models.supported`   | The models the line may resolve                                                               |
| `models.aliases`     | Tier-to-model mapping                                                                         |
| `effort.aliases`     | Effort-to-variant mapping                                                                     |
| `compaction`         | Mapping of the compaction window onto OpenCode's compaction keys                              |
| `preferred`          | Whether a bare `opencode` prefers line 2 when both lines are available; default true          |

A runner may carry either section, both, or neither. A section whose binary reports a version from the other line is a
configuration error, not a quiet promotion: the capability is unavailable and the runner's health names the line the
binary actually belongs to.

Both lines on one host need two binaries at two paths, and OpenCode 2's installer replaces a 1.x binary where it finds
one. The runner's install guide therefore places each pinned version in its own directory, from OpenCode's release
archives, and points each section at its own path; neither line's binary is the one on `PATH` unless the operator
chooses it for line 1.

## Store isolation

A line-2 store is a runner-owned OpenCode data, state, and cache directory. Isolation is by `XDG_DATA_HOME`,
`XDG_STATE_HOME`, and `XDG_CACHE_HOME` on every line-2 process the runner launches. `HOME` and `XDG_CONFIG_HOME` remain
the operator's, so the operator's global OpenCode configuration and skills are discoverable to line 2 as they are to
line 1, subject to what the proof records under ambient discovery.

The runner refuses a line-2 store that is the operator's personal OpenCode data directory, and one that holds a 1.x
store, because OpenCode 2 migrates a 1.x store in place on first start.

Line 1 keeps its sessions in the operator's personal store, which it does not isolate. On a host where the operator also
runs OpenCode 2 personally, that store can be migrated beneath the 1.x runner. Line 1's capability health therefore
recognizes a store OpenCode 2 has migrated and reports the line unavailable, naming the cause, rather than failing
session by session.

`blizzard runner opencode login --line 2` runs `opencode auth login --standalone` against the line-2 store, so the
operator authenticates the store once with their ordinary flow and the managed service is never started. Capability
health treats a store with no usable provider credential for the configured tiers as unavailable.

## Selection

An acceptable-set entry is a harness id with an optional line selector, `<id>[@<line>]`: `opencode` accepts any
available line, `opencode@2` and `opencode@1` only that line. The selector is additive to the acceptable-set grammar the
[harness selection contract](../../harness-selection.md) defines. Graph mint validates the grammar only: a line on a
harness that has none, such as `claude_code@1`, is well-formed and simply unclaimable, as any unsupported harness is.
Entries are unique as written, so `["opencode@2", "opencode"]` is valid and means "line 2 if there is one, else any
line".

A bare `opencode` on a runner holding both lines resolves to line 2 when `[opencode_2] preferred` is true and line 2 can
resolve the session's authored tiers, and to line 1 otherwise. A runner's default harness, used when the envelope
carries no set, names a line explicitly when it names `opencode`. Line preference applies only at mint.

A runner advertises one capability per available line, each carrying the harness id, the line, and the observed version,
and acquisition counts a line-selected entry as satisfiable only by a runner advertising that line. The hub stores
capabilities as JSON and ignores fields it does not know, so the line is additive on the wire. A runner from before this
epic advertises no line, and the hub reads its `opencode` capability as line 1, the only line such a runner can hold. A
runner from before this epic handed a line-selected entry treats it as a harness it does not hold and tries the next
entry, so a line-selected session never runs on a line it did not ask for.
