---
epic: live-config
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:live-config`

Adding a work source to a hub today is an edit to a file on the hub's own machine, followed by a restart. So is pointing
delivery at a different forge, and so is rotating the token it lands with, which sits in the hub's environment beside
every other secret it holds. For the operator of a hosted hub, saying something as small as "this hub should also read
issues from winter" costs a change to the infrastructure that deploys it and a redeploy. The configuration that
describes the fleet's work — where that work comes from, where it lands, and the credentials for both — is kept as if it
were configuration of the process, and every change to it pays the process's price.

This epic moves that configuration into the hub's store, where the hub can change it while it runs. It is the groundwork
`milestone:projects` stands on. `epic:multi-tenancy` and `epic:projects` each partition what the hub holds, and a work
source that lives in a file can be neither owned by a tenant nor grouped under a project. Making the move first, on the
hub as it is today, lets our own fleet prove the new source of truth before either grouping key exists, so the two epics
that follow add a key to tables that already exist rather than creating them, moving configuration into them, and
partitioning them in one stroke.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **Work sources as data.** Each work source the hub reads — its provider, its repository, whether it annotates, its API
  and web addresses — becomes a record that is created, edited, and retired through the API and the CLI, appears on the
  board, and takes effect without a restart. Sources and repositories take the verbs and the reversible retirement
  scopes and routines already carry: a record is retired, never deleted, so a source's history stays with the chunks
  that came from it.
- **The delivery target as data.** The forge address, owner, base branch, and token that delivery reads from the hub's
  environment today become repository records of their own. The hard-coded fallback owner goes with them, so a bare
  repository name resolves against a declared repository rather than a name baked into the hub. Repositories stay
  records of the whole hub — of the tenant, once `epic:multi-tenancy` lands — and `epic:projects` later lets each
  project link the ones its work lands in, so nothing here may assume there is only one.
- **A secret store the hub holds.** A credential is written once through the API and never read back, and is encrypted
  at rest under a key the hub's own configuration supplies. Work sources and repositories refer to a secret by name
  instead of naming a variable the hub resolves from its own environment, so rotating a token is an API call rather than
  a redeploy. The namespace is shaped so that `epic:multi-tenancy` can scope it to a tenant without renaming anything —
  once tenants exist, a reference into the hub's environment would let one tenant spend another's credential.
- **Configuration that can still live in a repository.** Moving the configuration into the store must not make it
  something that can only be clicked together. A declarative file, applied by the CLI, states the sources and
  repositories a hub should hold and reconciles the store to it, so an operator who versions their hub's configuration
  keeps doing so. It reconciles the way `graph sync` does, writing only what changed, and a dry run shows that
  difference before anything is written. It reconciles strictly: a record the file once declared and no longer names is
  retired, and while a file owns a record, every other door refuses to change it until the record is released from the
  file.
- **Configuration you can see.** The board gains a place to look at what a hub is configured with: its sources and
  repositories, its secrets by name and never by value, what each record last changed and who changed it, which records
  are retired, and which a declarative file manages. It is designed as mockups before it is built and proven by looking
  at it, because a configuration screen that reads wrong is a configuration screen nobody trusts. The mockups —
  [desktop](./artifacts/configuration-desktop.html) and [mobile](./artifacts/configuration-mobile.html) — put it under
  Admin beside today's users page, one surface per configured noun plus a log of every change, and carry a switch that
  compares the board editing configuration against only showing it.
- **One way to change a configured thing.** The hub already has several, and they disagree. Editing a routine restates
  every field and calls it a patch; editing a work item replaces only the fields it names; editing a scope changes its
  one field. Graphs arrive as YAML while the hub's own file is TOML. Every configured record this epic adds would
  otherwise pick a side at random, and every client — the CLI, the board, a declarative file, `epic:terraform-provider`
  — would learn each record's habits one by one. So the epic settles a single convention and writes it into
  blizzard-context's architecture, as the shared understanding every configured surface is built and reviewed against:
  the verb set a configured record carries, what a patch means, the file format declarative ingestion reads, what an
  apply and its dry run promise, retirement in place of deletion, and secrets that are written but never read. Every
  configured surface keeps it by the time the epic is done: the records this epic adds, and the routines, scopes, work
  items, and graphs that predate it, brought into line within the epic rather than excused — a rule the code does not
  already keep is a rule no review can hold anyone to.
- **The hub that exists today, carried over.** On its first start after the change, the hub imports its file's work
  sources and its environment's delivery target into the store and writes their tokens into the secret store. From then
  on it refuses to start while the old keys remain, so no setting ever has two owners.

## What stays in the file

The hub's file keeps what the hub must know before it has a store to read or a person has logged in: where its store
lives, where it listens, which proxies it trusts, how people log in, the rollout modes for runner authentication, route
tokens, and produced artifacts, the transcript caps, tracing export, the annotation cadence, and the switch that keeps a
non-canonical hub from closing real items. None of them belongs to any one tenant or project, and none moves here.

## How a change takes effect

An operator who adds a source or rotates a token expects the next thing the hub does to use it — not the next restart,
and not whatever a process remembered from an hour ago. So the hub holds no configuration in memory as a source of
truth: a sweep reads its sources at the start of each pass, a delivery step reads its repository when it runs, and a
request reads what it needs. The store is the only answer to what is configured, which keeps the hub correct whether one
process asks it or several.

What is worth keeping is what configuration is turned into — an authenticated forge client, a work-source adapter, a
decrypted token. Each configured record carries a revision that moves with every change, and anything built from a
record is kept against the revision it was built from, so the first use after a change rebuilds it without any process
having to tell another. A rotated token is a change like any other, and takes effect on the very next call, mid-chunk
included.

Nothing is pinned to a chunk. Work already in flight sees a change the next time it uses the record. A change that would
surprise that work — pointing a source at a different repository, moving a base branch — is rare and deliberate, and
whoever makes it pauses the work it affects first. Retiring a source or a repository stops new work from it; chunks
already taken from it finish.

## The API is the contract

The CLI and the declarative file are the first consumers of the configuration API, not the last;
`epic:terraform-provider` is the second, and the API is built to make that provider thin. Every configured record has a
stable identifier and a full set of verbs — create, read, edit, retire — and retirement stands where a deletion would.
Writes are idempotent, and a caller can see the difference a write would make before it makes it. A secret is written
and never read back, so no client, file, or state ever holds one it did not send.

## Open questions

- Where the secret store's key comes from and how it is rotated — the hub's environment, a file, or an external key
  service — and what an operator does when it is lost.
- How the split between file and store sits within the single resolution order `epic:config` declares for every setting.
- Whether the board edits configuration as well as showing it, or leaves changes to the CLI and the declarative file;
  the mockups are where this is decided.
