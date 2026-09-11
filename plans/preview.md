# Plan — `epic:preview`

A reviewer reads a diff and imagines the feature. That is the whole of what blizzard offers today, and it is a poor
substitute for opening the thing and using it — particularly for work whose value is in how it feels. Meanwhile the
environment the chunk is working in is alive, running, and has the branch checked out. It is simply unreachable. This
epic makes it reachable: the operator opens the running application from the board and judges it the way a user would.

The unsolved half is reachability, and it is the epic's real content.

## What to build

- **A path to a live workspace that costs the trust boundary nothing.** The hub never reaches into a runner, which is
  precisely what lets a runner sit on a laptop behind NAT. Of the available shapes — a tunnel the runner opens and the
  hub multiplexes, an operator-supplied private network, or a published runner — only the first preserves that, and it
  is the one to build.
- **The workspace seam says what it is serving.** An optional capability, for providers that run services at all: a
  workspace names the address of the application it is hosting, and a provider that hosts nothing declines rather than
  pretends.
- **The affordance on the chunk.** A live chunk offers its environment where the operator is already watching it; a
  finished one does not offer what no longer exists.
- **A preview that dies with its chunk.** The environment's lifetime is the chunk's, and the platform is explicit about
  that rather than leaving a link that quietly stops working.

## Open questions

- The tunnel's transport and how a session is authorized, since an opened tunnel is a hole in the boundary the design
  exists to protect.
- What a provider that runs no services offers instead, if anything.
- Whether a preview is ever shareable beyond the operator, which is a small feature and a large security question.
