# Plan — `epic:provider-kit`

Blizzard's seams are real, and every binding behind them ships inside the wheel — so someone who wants their own work
source writes a pull request against blizzard and waits. Criterion 5 is proven at two live bindings, and blizzard cannot
honestly claim the seam is a seam until one of them comes from outside. This epic makes a binding something a stranger
can write and ship: an interface they can read, discovery that finds what they installed, and a suite that tells them
they got it right before a chunk depends on it.

Work sources and workspaces come first — the two seams whose next occupants are likeliest to come from outside.

## What to build

- **A declared interface.** What a binding must provide and what it may rely on, written for someone who does not read
  blizzard's internals — which is the test of whether the seam was ever really one.
- **Discovery without a code change.** An installed binding is found and usable without editing blizzard, which is the
  difference between a plugin and a patch.
- **A conformance suite.** A binding proves itself before it runs a single chunk, so the author learns about a
  misunderstanding from a test rather than from a wedged night.
- **One binding moved out of the tree**, to prove the path is walkable by someone who is not us.

## Open questions

- The discovery mechanism, and how much of the host language's packaging conventions blizzard should adopt rather than
  invent.
- What the suite can genuinely assert versus what it can only smoke-test — a work source that lies about its items will
  pass anything cheap.
- How the interface is versioned, and what an older binding does when it meets a newer hub. Delivery and the coding
  harness stay in-tree for now, carrying the fencing and execution guarantees; the human channel joins once `epic:chat`
  builds its fan-out seam.
