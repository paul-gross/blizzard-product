# OpenCode major versions specification

OpenCode 1 and OpenCode 2 are two major versions of one harness, `opencode`. Each has its own binding, admitted
versions, and fixture corpus. They share the harness id and the runner's configuration: which major version a runner
works is decided by the version of the binary it is configured with, never by a setting.

## Binding dispatch

A runner configures one `opencode` binary, and its observed version selects the binding. A version inside OpenCode 1's
admitted range composes the OpenCode 1 adapter, one inside OpenCode 2's admitted range composes the OpenCode 2 adapter,
and any other version is version-incompatible and refused, as today. A runner therefore holds at most one major version
at a time, and advertises one `opencode` capability carrying its observed version, exactly as it does now. Nothing the
hub or the mock hub keys by harness id changes: eligibility, acquisition, and the wire see one capability per harness,
and a runner from before this epic is indistinguishable from one that holds OpenCode 1.

The binding is composed at daemon start, as the runner's harness registry is today, and never swapped while the runner
runs. The binary's version selects it. The version the runner advertises, records on every spawn, and compares before
every launch is the service's own, as its API reports it, and the binary's where the API reports none. A launch whose
observed version differs from the advertised one — the operator's service updated itself, or the binary was replaced —
is refused before anything starts and triggers a fresh probe; a probe that finds a version still inside the composed
binding's range makes it available again under the new version, and one that finds another major version, or a binary
and service in different major versions, leaves the capability unavailable, naming the change, until the runner is
restarted.

A machine runs one major version. Moving it to OpenCode 2 is the operator's sequence to carry out: stop the runner,
upgrade OpenCode, start the runner. Running OpenCode 1 and OpenCode 2 runners side by side on one machine is not
supported.

## Sessions

The OpenCode 2 binding resumes only a session whose metadata names its runner. A 1.x session — whether still in a 1.x
store or migrated into OpenCode 2's by OpenCode 2 itself — carries no such metadata, so it is missing and takes the
existing path for a lost conversation. Nothing tracks which major version a session belongs to. A runner moved from 1.x
to 2.x loses its open 1.x conversations, which are ephemeral, and its work carries on in fresh ones. The harness version
recorded on every spawn, as the [harness selection contract](../../harness-selection.md) requires of every harness,
stays a record for analytics and diagnosis, never a dispatch key.

## Admission

| Major version | Admitted versions | Corpus                                                                  |
| ------------- | ----------------- | ----------------------------------------------------------------------- |
| OpenCode 1    | `>=1.18.25,<2.0`  | `src/blizzard/runner/harness/contracts/opencode/1.18.25/`               |
| OpenCode 2    | `>=2.0,<3.0`      | `src/blizzard/runner/harness/contracts/opencode/2.1.4/`, from the proof |

Each major version admits a range, not a set of proven releases. A release inside the range is trusted until it breaks,
and a break is fixed when it appears: the proof is rerun against the release that broke, its corpus is committed beside
the first, and the binding is corrected against it. Both major versions' corpora live side by side under
`src/blizzard/runner/harness/contracts/opencode/`, one folder per proven release, and each major version's corpus guards
read only the folders inside its own range. OpenCode 1's range is not changed by this epic.

## Runner configuration

The `[opencode]` section is unchanged, and an operator who moves a runner from 1.x to 2.x changes its `binary` and
nothing else. `enabled`, `binary` (including one found on `PATH`), `models.supported`, `models.aliases`, and
`effort.aliases` mean the same thing on either major version, and OpenCode 2 carries effort into the model reference's
variant. `auth_path` and `worker_config_path` have no meaning on OpenCode 2, which uses the operator's own
authentication and injects no worker configuration, so its binding ignores both. The compaction window is not
configuration; it arrives with each lease, and the OpenCode 2 driver enforces it.

## The operator's store

OpenCode 2 sessions live in the operator's own store, on the operator's own service, under the operator's own
authentication; the runner owns no store and has no login of its own
([execution](./execution.md#the-operators-opencode-service)). Capability health treats a service with no usable provider
credential for the configured tiers as unavailable.

OpenCode 1 also keeps its sessions in the operator's personal store, and OpenCode 2 migrates a 1.x store in place the
first time it opens it. An OpenCode 1 runner whose store has been migrated beneath it — by an operator who upgraded
OpenCode without stopping it — reports its capability unavailable, naming the cause, rather than failing session by
session.

## Selection

Graphs, chunks, and runner defaults name `opencode` and nothing more; the acceptable-set grammar is unchanged. A runner
offering `opencode` offers whichever major version its binary is, and a fresh mint takes that major version. A fleet
that wants a lineage on one major version in particular routes it the way it routes any work to particular runners, not
through the harness id.
