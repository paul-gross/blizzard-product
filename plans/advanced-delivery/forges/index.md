# Plan — `epic:advanced-delivery`, forges slice

Every script that lands work today carries the keys to the forge it lands on. The hub reveals a repository's token and
hands it to the delivery script as an environment variable, and the script speaks GitHub's API directly, path by path.
That costs three things at once. The token reaches a process that needs only a handful of operations on one repository,
yet could do anything the token can. A forge is known only as an address repeated on every record that touches it, so
the hub cannot say which records share a server, or speak to one in any voice but GitHub's. And a script written for one
forge's vocabulary has no way to run against another, so every policy the rest of this epic promises would be written
once per forge.

This slice gives the forge a name and a seam. An operator declares each forge server their tenant uses once, and every
repository and every forge-backed work source names the forge it lives on. A landing script holds no token: it asks the
hub, the hub acts on the forge with the repository's own credential, and both speak one vocabulary of refs, changes, and
checks that no forge's own words leak into. A script says which of the forge's capabilities it cannot do without, and
the hub refuses the landing before anything runs when the repository's forge lacks one, rather than letting the script
discover it halfway through a merge.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **A forge, declared once per tenant.** A forge is a configured record of the tenant, known as a project is known: by
  an id that never changes, a short slug people type and may change, and a display name. It carries its kind and the
  addresses of its API and its web pages. It is created, edited, retired, and re-enabled through the same doors as every
  other configured record — the admin screens, the CLI, and the configuration document — and its changes appear in the
  same change log.
- **Repositories and forge-backed work sources name their forge.** A repository stops carrying a forge address of its
  own and names a forge instead; so does a work source whose provider is a forge. Each keeps its own credential, so the
  key that reads a tracker's issues and the key that lands into a repository can differ, as they can today. A forge that
  an enabled record still names cannot be retired.
- **One landing seam.** The hub speaks to a forge through one seam that reads a branch's tip, compares two revisions,
  opens, reads, updates, and merges a change, reads and re-runs the checks on a revision, and confirms that a commit
  exists. Each forge kind answers in the seam's own vocabulary, so a script reads that a change is behind its base or in
  conflict, never GitHub's `mergeable_state`. GitHub is the one kind this slice builds; the hub's other forge readers,
  such as the garden's check that a delivered commit exists, move onto the same seam.
- **Landing through the hub.** A delivery script holds no forge token and no forge address. It reaches the forge only
  through a callback on the hub, authorized by the same per-visit token its marker writes already carry, and only for
  the repositories its chunk's commits resolved to. The hub performs each operation with the repository's own
  credential.
- **Capabilities a script can count on.** Each forge kind declares the capabilities it offers. A graph's delivery node
  names the ones its script cannot do without, and a landing into a repository whose forge lacks one is refused before
  any step runs, naming the capability and the forge. A capability a script can live without is not named; the script
  reads what the forge offers and takes its own fallback, as today's policy can carry on without re-running a check.
- **The packaged policies, ported.** The landing scripts blizzard ships are rewritten against the seam and behave as
  they do today, so the fleet lands exactly as it did before, without a token in sight.
- **Carried over without a hand on it.** An installation becomes one forge per server its records already name, each
  repository and work source pointed at its own, with nothing reconfigured by hand.
- **Old fields through a transition, then retired.** For a release, every door still accepts the addresses a repository
  and a work source carried before, resolving each to its forge, and a delivery script of an operator's own still
  receives the token it was written for, so a configuration document, a runbook, or a script written before the slice
  keeps working while it is moved over. The hub notes every use and warns whoever made it. Once the uses stop, the old
  fields and the token are retired, and anything still sending them is refused with a message naming what replaced it.

## Left for later

Everything the seam does not yet need waits for the slice that needs it. A second forge kind, GitLab, arrives with the
`policies` slice, where a chunk first lands across forges and the seam's vocabulary first has to hold for two of them;
so do the forge's webhook and the rate budget its callers share, which matter once parked changes are watched at scale.
A capability that varies from one repository to another, such as a merge queue switched on for one repository and not
its neighbour, arrives with the `trains` slice, and until then a forge's capabilities are its kind's. Authenticating as
an application rather than with a token waits for an operator who needs it.
