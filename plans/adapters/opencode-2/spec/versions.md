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

The version is read where the runner reads it today: on every capability probe, which chooses the binding, and before
every launch. A launch whose observed version differs from the version the capability advertised — an operator ran
`opencode upgrade`, or replaced the binary — is refused before anything starts, marks the capability unavailable, and
triggers a fresh probe, which composes whichever binding the new version selects. OpenCode 2's service is the operator's
and may update itself; the comparison is what catches it, against the version the service reports.

An operator who wants both major versions on one machine runs two runners, each configured with its own binary. The
runner host already hosts several runners; the two major versions need nothing more from it. Both major versions install
a binary named `opencode`, and OpenCode 2's installer replaces a 1.x binary where it finds one, so the runner's install
guide places each major version's binary in its own directory, from OpenCode's release archives, and points each
runner's `binary` at its own path.

## Sessions

A session lives in the store its major version keeps — OpenCode 1's and OpenCode 2's are separate — and the two never
share a session. Nothing tracks which major version a session belongs to or checks it: a binding asked to resume a
session its store does not hold finds it missing, and the attempt takes the existing path for a lost conversation. A
runner moved from 1.x to 2.x simply loses its open 1.x conversations, which are ephemeral, and its work carries on in
fresh ones. The harness version recorded on every spawn, as the [harness selection contract](../../harness-selection.md)
requires of every harness, stays a record for analytics and diagnosis, never a dispatch key.

## Admission

| Major version | Admitted versions | Corpus                                      |
| ------------- | ----------------- | ------------------------------------------- |
| OpenCode 1    | `>=1.18.25,<2.0`  | `contracts/opencode/1.18.25/`               |
| OpenCode 2    | `>=2.0,<3.0`      | `contracts/opencode/2.1.4/`, from the proof |

Each major version admits a range, not a set of proven releases. A release inside the range is trusted until it breaks,
and a break is fixed when it appears: the proof is rerun against the release that broke, its corpus is committed beside
the first, and the binding is corrected against it. Both major versions' corpora live side by side under
`contracts/opencode/`, one folder per proven release, and each major version's corpus guards read only the folders
inside its own range. OpenCode 1's range is not changed by this epic.

## Runner configuration

The `[opencode]` section is unchanged, and every key in it means the same thing on either major version: `binary`,
including one found on `PATH`, `worker_config_path`, `models.supported`, `models.aliases`, `effort.aliases`, and
`compaction`. Each binding applies a key the way its major version needs it — OpenCode 2 carries effort into the model
reference's variant, enforces the compaction window itself rather than through OpenCode's configuration, and has no
worker configuration to inject, so it leaves `worker_config_path` unused — but an operator who moves a runner from 1.x
to 2.x changes the binary and nothing else.

## The operator's store

OpenCode 2 sessions live in the operator's own store, on the operator's own service, under the operator's own
authentication; the runner owns no store and has no login of its own
([execution](./execution.md#the-operators-opencode-service)). Capability health treats a service with no usable provider
credential for the configured tiers as unavailable.

OpenCode 1 also keeps its sessions in the operator's personal store, and OpenCode 2 migrates a 1.x store in place the
first time it opens it. On a host where an OpenCode 1 runner and OpenCode 2 share a user, the 1.x store is migrated
beneath the 1.x runner. OpenCode 1's capability health therefore recognizes a store OpenCode 2 has migrated and reports
the capability unavailable, naming the cause, rather than failing session by session.

## Selection

Graphs, chunks, and runner defaults name `opencode` and nothing more; the acceptable-set grammar is unchanged. A runner
offering `opencode` offers whichever major version its binary is, and a fresh mint takes that major version. A fleet
that wants a lineage on one major version in particular routes it the way it routes any work to particular runners, not
through the harness id.
