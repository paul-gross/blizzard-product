# Plan — `epic:config`, model-resolution slice

A graph author writes a capability tier and an effort, never a model, and that is what lets one graph run under either
harness. The price is that the model a session actually runs on is decided at a distance, in several places that each
have a say: the graph's session pool pins a tier and an effort; the chunk may carry defaults of its own; the runner's
configuration maps each tier to a concrete model per harness and renames efforts on the way through; and each adapter
brings built-in defaults underneath all of it, which quietly answer for any tier the runner leaves unmapped.
[`persona:harness-engineer`](../../../charter/personas/harness-engineer.md) is the person this lands on. Their craft is
moving one of those settings and reading what changed, and today the only way to know what a pool will run on is to hold
the graph, the runner file, and the adapter defaults open side by side and do the resolution in their head — the same
reconstruction they would be doing the morning after a bad night, only in advance.

The runner should answer that question itself. One command, on the machine the runner lives on, prints the resolution
the way the runner will perform it: every layer that holds a value, which value won, and why. It is the same answer
`epic:config` promises across the whole configuration surface — a lookup rather than an investigation — given early to
the one corner of it where the answer changes what a night costs.

## What to build

- **The tier table.** For each capability tier and each harness this runner has configured, the adapter's built-in
  mapping where one exists, the runner's own alias where the operator has set one, and the model that results. A tier a
  harness cannot resolve is shown as exactly that, since it is the reason a runner will pass over work demanding it.
- **The effort table.** The same shape for effort: the value as authored, the runner's alias where one renames it, and
  what reaches the harness — including a value the harness will drop rather than honor, which today fails silently.
- **The session pools.** For the graphs this runner works, each declared session's harness set, model preference, and
  effort, resolved through both tables to what each harness in its set would actually run, with the chunk default shown
  where it is the layer that supplies a field.
- **Provenance on every cell.** Each resolved value names the layer it came from — adapter default, runner alias, graph
  pin, chunk default, or adapter fallback — so an operator scanning the table sees at a glance which answers they chose
  and which were chosen for them.
- **Two readers, one answer.** A table a person reads at the terminal, and a JSON form an agent or a script reads, each
  carrying every layer's value alongside the effective one rather than the result alone. Both come from the resolution
  the runner performs at spawn, never from a second implementation of it that could drift.

## Where this stops

The command reports; it changes nothing. It reaches no web surface and no board view: a place to look at this in the
browser is a later decision, and a read-only CLI is the right first home for a question asked mostly by the person
editing the files it describes. It stops at the session a worker is spawned into. The models of the subagents that
session goes on to spawn are set by the workspace the worker runs in, not by blizzard, and blizzard only observes them
in the transcript; that half of the picture is winter's to report. Joining the two into one view — blizzard and winter
configured and inspected together — is a larger question about how the two products meet, and it is not asked here.

## How the answer is assembled

The layers live in two places, and the command respects that split rather than papering over it. The tier and effort
tables belong wholly to the runner, so they answer from the machine alone, with or without a hub. The graphs and chunk
defaults live on the hub, and so does the rule that merges them: the hub already decides, field by field, whether a
session's model comes from the graph or the chunk. The runner asks the hub for that merge, with each field's source
attached, instead of re-deriving it from graph files that may not match what was minted. When the hub is out of reach,
the command says so and still prints everything the runner owns.

The pools follow selection all the way through. Every harness in a session's acceptable set is listed with what it would
run, and the one the runner would actually choose is marked, with the reason each earlier harness was passed over. A
table that stopped short would hand the operator the last and most error-prone step of the reconstruction this command
exists to spare them.

Chunk defaults appear only when the operator names a chunk. Without one, the pools read as the graphs declare them,
which is the view that holds for every chunk. Naming a chunk also brings in what that chunk is running right now on this
runner: a pool already alive keeps the model it was minted with, so an alias edited this morning changes the next
session, not the current one, and the operator should see both side by side rather than discover the difference in a
transcript.

| Where                                             | Read when                                                                   |
| ------------------------------------------------- | --------------------------------------------------------------------------- |
| [spec/](./spec/index.md)                          | Implementing any part of the slice or resolving its technical contract      |
| [sample](../artifacts/model-resolution.txt)       | Seeing what the operator reads at the terminal, on one worked configuration |
| [sample JSON](../artifacts/model-resolution.json) | Seeing the same answer as a script or an agent reads it                     |
