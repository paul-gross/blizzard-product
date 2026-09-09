# Plan — `epic:fact-egress`

An operator who wants to know what last week cost already has the numbers and still has no way to hold them.
`epic:analytics` made the fleet's facts reachable — every file a worker opened, every agent it spawned, every step's
duration and price — and gave an analyst the verbs to pull them from a terminal. What it never did was let anyone bring
their own tool. The answer arrives once, assembled by a script written for that one question and discarded after it, and
seven days later the same question costs the same afternoon.

The tools that would answer it in a second already exist. Metabase, Superset, Grafana, DuckDB, and whatever displaces
them in three years are mature, most of them are free, and not one of them can reach a blizzard fact today. The gap is
not a missing chart. It is that blizzard's data has no exit.

## What blizzard owes, and what it does not

The reflex, when people want dashboards, is to build dashboards. That reflex is wrong here and expensive to indulge:
every chart blizzard draws is a chart blizzard maintains, and it will still be the wrong chart for somebody. An operator
running the fleet against their own project arrives with their own warehouse, their own tools, and their own idea of
what Monday should look like. A team without any of that will pick tools blizzard has never heard of. Neither is served
by an opinion baked into the platform.

So blizzard owes them an exit, not an interpretation. The facts leave as files, in a shape the tools already read, at a
location the operator names. Which warehouse receives them, which dashboard renders them, which loader moves them —
those are the operator's decisions, and blizzard never learns the answers.

That boundary is what makes this a capability rather than a feature, and it is the same bargain the charter strikes
everywhere else: a named seam, an interface blizzard honors, and a binding that belongs to whoever is running the thing.
The measure of success is not a screenshot. It is that someone wires a tool blizzard has never been tested against, in
an afternoon, and needs nothing from us to do it.

## What stands between a fact and a file

Three things, each small alone, and together the whole of the work.

The first is **identity**. Every rollup today groups by node id, and a node id is minted afresh each time a graph is
minted — so the single station an operator calls "build" is scores of unrelated keys, and anything downstream is left to
reassemble them against every graph version the hub has ever held. No axis can be drawn through that. A chart's
dimensions are made of names.

The second is **time**. The operational reads accept a window as a filter but carry no timestamp on the row, so "cost
per day" is not a slow query — it is one request per day, with the shape of the week reassembled by whoever asked. A
timestamp on the fact is what turns a filter into an axis.

The third, and the one that decides everything else, is **grain**. Every endpoint returns a sum, and a sum cannot be
taken apart: once the hub has added up cost by node, nothing downstream can ask that same answer for cost by node *and*
week. Under a capability this is not merely inconvenient, it is a broken promise — every aggregate blizzard pre-computes
is a question the operator's tool is forbidden to ask. The facts must leave one level below the aggregate, their
dimensions spelled out, so that summing happens where the question is.

## What to build

**A normalized fact stream.** One row per completed step, carrying its chunk, its graph and node by name, its source,
its start and end, its duration, the choice it resolved to, and its tokens and cost. One row per derived event, carrying
its kind, subject, tool, agent type, depth, and the node context it belongs in. Every row self-sufficient — every
dimension it could ever be grouped by present on the row itself, rather than waiting to be joined back from somewhere
else. Denormalization is not a shortcut here; it is what lets a tool with no join support answer the same questions as
one with it.

**An exporter that writes files.** Parquet by preference, because it carries its own schema and lets a reader scan only
the columns a question names; NDJSON where a plain-text stream is easier to consume, with a manifest supplying the
schema Parquet would have carried itself. The destination is configured — a local directory, an object-storage prefix —
and blizzard writes there and stops. No credentials to a warehouse, no vendor SDK, no loader. The tools that move files
between systems already exist and are better at it than blizzard would be.

**An incremental cursor.** The facts are append-only and already carry monotonic keys, so an export resumes where the
last one ended rather than rewriting history every night. The cursor is persisted, so a restart costs nothing, and
re-shipping a row twice is harmless by construction.

**A versioned contract.** The exported shape is a published interface, not an accident of the store's current schema. It
lives with blizzard's other wire contracts, it carries a version, and it changes on a deprecation path — because the
moment someone's dashboard depends on a column, an internal refactor that renames it is a broken promise rather than a
tidy-up.

**A data dictionary.** What each column means, what a null means, what the enumerated values are. A capability whose use
requires reading blizzard's source is not a capability; it is a puzzle with a support burden attached. For this epic the
documentation is a deliverable, not a follow-up.

## What must never be true

**The export can never cost the fleet its liveness.** The charter promises crash-equivalence — a kill at any instant
loses at most in-flight tokens — and an exporter sitting in the hub's critical path would trade that promise away to a
storage backend having a bad afternoon. Facts land in the store the way they already do; a separate sweeper reads them
forward and writes files. Nothing in the fleet's path ever waits on the exit.

**Content never leaves by accident.** Transcripts hold prompts, file contents, and whatever a worker happened to read,
and once a byte reaches somebody's warehouse it is beyond recall. What leaves is decided field by field rather than
table by table, and the default is that dimensions and numbers go while content stays. An operator who wants more says
so deliberately.

**A fleet with no sink configured behaves exactly as it does today.** The capability is off until someone turns it on,
and its absence costs nothing.

## What this epic is not

It is not a dashboard, and it is not a second product. Blizzard renders nothing here and hosts nothing here.

It is not a set of vendor adapters. There is no S3 exporter and no BigQuery exporter and no Postgres loader, and adding
one later would be a mistake — the ecosystem of tools that move files into warehouses is large, mature, and none of
blizzard's business.

It is not a query language. Blizzard's read surface keeps its enumerated routes and its filter vocabulary. What changes
is that it also emits facts, un-summed, for someone else's engine to aggregate.

And it is not an interpretation. Deciding that a gate which never fails is a gate not worth running remains a person's
judgment. This epic's job ends where the numbers become legible.

## How it is proven

The charter already says how a seam earns belief: the promise holds the first time the seam has two live bindings. So
the acceptance is not a chart that looks right. It is two independent tools — a warehouse-backed BI surface and a local
engine reading the files directly — wired against the export from the outside, using only the published dictionary and
no blizzard change between them. `blizzard-mock` supplies the fleet, so the proof costs no real transcripts and no
vendor quota. If either wiring takes longer than an afternoon, the capability is not finished.
