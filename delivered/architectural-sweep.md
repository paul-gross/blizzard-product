---
epic: architectural-sweep
refinement: pristine
slices:
  - name: full
    status: delivered
---

# Plan — `epic:architectural-sweep`

Most of blizzard is built by agents that have never seen it before. A fleet agent picking up a chunk finds the code for
a behavior from the behavior's name, or it does not find it at all and guesses (`persona:fleet-agent`). An architect
deciding where a new rule belongs reads the layout for the answer (`persona:application-architect`). Today the layout
answers neither of them. The hub's domain is sixty-odd modules in one directory, with no line between one concept and
the next. The runner is sliced by what code does rather than what it is about, so a single idea like provider overload
lives in three packages at once. The code still passes every gate, and that is exactly the trouble: nothing tells anyone
the structure has drifted until a change that should touch one place touches six.

This epic makes the structure say what blizzard does, and keeps it saying so. Every concept in both daemons and the
board gets one place named for it. Within each daemon, packages depend on one another in one direction only, so the
ground floor never reaches up into the rooms built on top of it. Every data class says what kind of thing it is — a
domain model that carries rules, a database row, or a plain carrier of data between two places. And every business rule
moves onto the model it governs, leaving the services around it to do only the sequencing.

## How it lands

The work is nine sweeps, landed one after another and never side by side, because each reshapes ground the next one
builds on:

1. Every data class declares its role, so later sweeps know which types are the models.
2. The hub's domain is regrouped into concept packages with a one-way dependency graph.
3. The runner's loop stops handing every step the whole world, and each step asks only for what it uses.
4. The runner's harness adapters move into packages of their own, behind a single module that knows them both.
5. The runner is regrouped into concept packages with its own one-way dependency graph.
6. Business rules move onto their models across both daemons, now that every model has its final home.
7. The wire becomes the board's complete domain model, so the board stops re-deriving what the backend already decided.
8. The board is regrouped into feature folders with its own one-way dependency graph.
9. The board's containers stop deciding things, and what they derived moves into small pure models beside each feature.

Each sweep is a single change that reshapes all of the code it governs and, in the same delivery, writes down the rule
and adds the test that enforces it. A rule never exists before the code obeys it, and once it lands, the test refuses
any change that would break it. That is what keeps the result from eroding: the structure is held by the build, not by
anyone's memory.

## Out of scope

Moving each concept's stores, API routes, and CLI verbs into its package is the end state this points toward, and is
left for later. `blizzard-mock` is untouched.
