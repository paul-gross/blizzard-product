# Plan — `epic:chokepoint`, epic-dwf slice

Every lane the fleet has today is built around one agent at a time. A chunk travels its graph node by node, and at each
node a single session does the work, however large the work turns out to be. That suits the feature-sized change most
chunks carry. It does not suit the change a project makes a few times a year: rewriting a component from Python into
Rust, moving the board from one framework to another, upgrading a generation of dependencies, or clearing out a body of
tech debt across the whole codebase. The work behind `epic:architectural-sweep` was one of these. It reached its tens of
thousands of lines because dozens of agents worked in parallel against one plan, and it was driven by hand from outside
the fleet, because no graph blizzard ships could have carried it.

This slice gives that work a lane of its own: a graph, `epic-dwf`, built for epic-level development. It sits beside
`bas-dwf` and `adv-dwf` as the heaviest of the development workflows, and the work it is meant for comes in two shapes.

The first is **epic-level development** — a change measured in twenty thousand lines or more, too large for one session
to hold and too interlocked to split into separate chunks that land on their own. Here the graph plans the change as a
whole, divides it into pieces that can be built side by side, sets a swarm of agents to them at once, and brings their
work back together into a single change that is verified, reviewed, and landed as one.

The second is **the swarm of small adjustments** — work that is broad rather than deep. A dependency bump that ripples
through four hundred call sites, a new rule that every module must now obey, a rename carried through the whole
codebase: no single edit is hard, but there are thousands of them, and each is best made by an agent looking only at its
own corner. Here the graph's value is less in the planning than in the fan-out, and in proving afterwards that the sum
of many small edits still behaves like the system it replaced.

Both shapes share the property that made the architectural sweep safe: they can only run when nothing else is moving. An
`epic-dwf` chunk is, in practice, always a [chokepoint](./chokepoint.md), and the two are meant to be used together —
the chokepoint clears the road, and the swarm uses all of it.

## How the lane runs

The swarm lives inside a single node. One session takes the chunk, reads the documents that govern the work, and runs
the swarm from there as a dynamic workflow — planning, fanning out, verifying, reviewing, and assembling the change
without handing the chunk back to the graph until it is ready to land. That is how the architectural sweep was
delivered, and the account its orchestrating session wrote afterwards,
[Nine Sweeps, One Line](./artifacts/nine-sweeps-one-line.md), is this slice's reference. The node's prompt is molded
from it, practice by practice: disposable agents that hand off before their context runs long, scripted moves kept apart
from judgment so they can be replayed onto a base that moved, every new rule paired with a gate that enforces it and a
cold evaluation that proves agents can find it, each reviewer paired with a skeptic told to refute it, and an itemized
audit of every decision the human made, so that nothing quietly becomes a follow-up.

That changes the shape of the graph. The traditional lanes go back and forth by design: a plan bounces off its gate,
review sends the build round again, and every station is a separate session so that its judgment stays independent of
the work it judges. An `epic-dwf` chunk does not travel that way. Its review is designed by the session that ran the
build, at a scale no single review node could reach, and runs inside the swarm alongside everything else. The graph
around the swarm is short: the node, the questions it raises, and the landing.

The operator enters at two moments, both on purpose. Before any code moves, the swarm surveys the change and gathers
every decision the documents cannot settle into one list, so the operator answers them together rather than meeting them
one at a time through the night — the sweep turned a hundred and forty-eight hidden decisions into ninety-six questions
answered in a single reply. And before the change lands, the swarm stops and asks, because work this size goes out under
a person's name.

Work reaches the lane only by choice. Triage never routes a chunk here: an operator who sends work down this lane is
also deciding to stop the fleet for it, and that is a decision a person makes deliberately, not one a classifier makes
on their behalf. Choosing `epic-dwf` and marking the chunk a chokepoint are the same act.

The lane closes with a retrospective, the way `adv-dwf` does, and here it carries more weight than anywhere else. The
reference records one delivery, and some of what worked there belonged to that epic rather than to the lane. Each run's
retrospective writes its own account of what the swarm did, what held, and what it would change, and the node's prompt
is molded again from what those accounts have in common. The lane is meant to get better at being a swarm every time it
runs.

## The harness

The node runs on a harness that can orchestrate a swarm of its own, and today that means Claude Code, whose dynamic
workflows are what drove the architectural sweep. The requirement is the capability, not the vendor: the node declares
that it needs a swarm-capable harness, and `milestone:polyglot` already lets a node choose its harness. When the slice
is picked up, its first piece of work is research — whether Codex, OpenCode, or any other harness blizzard runs can now
orchestrate parallel agents inside one session, and well enough to carry a change this size.

## Open questions

- What the swarm may spend, and where the line is drawn. Today's spend controls were built for chunks that move from
  node to node. A chunk's cap is checked at the boundary between one node and the next, and a runner's rolling ceiling
  pauses the runner from claiming more work without stopping the session already running. A swarm spends its whole
  budget inside one node, over the better part of a day — the architectural sweep ran some four hundred and fifty agents
  — so neither control would interrupt it once it started. The lane needs a ceiling that reaches inside the node, set by
  the operator before the swarm starts and honored while it runs.
