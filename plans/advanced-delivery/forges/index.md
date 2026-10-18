---
epic: advanced-delivery/forges
refinement: refined
status: horizon
---

# Plan — `epic:advanced-delivery/forges`

Once the `actions` epic lands, landing is the hub's own `land` action, and it still speaks GitHub and only GitHub. It
builds its requests path by path from an address the repository repeats, and reads GitHub's own words for whether a pull
request is mergeable or a check has passed. A forge is known only as that address, repeated on every record that touches
it, so the hub cannot say which records share a server, or speak to one in any voice but GitHub's. And every policy the
rest of this initiative promises would have to be written once per forge.

This epic gives the forge a name and a seam. An operator declares each forge server their tenant uses once, and every
repository and every forge-backed work source names the forge it lives on. The hub speaks to every forge through one
seam, in one vocabulary of refs, changes, and checks that no forge's own words leak into, and `land` speaks only that.
An action says which of the forge's capabilities it cannot do without, and the hub refuses the landing before the action
runs when the repository's forge lacks one, rather than letting it discover the gap halfway through a merge.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **A forge, declared once per tenant.** A forge is a configured record of the tenant, known as projects, repositories,
  and work sources already are: by an id that never changes, a short slug people type and may change, and a display
  name. It carries its kind and the addresses of its API and its web pages. It is created, edited, retired, and
  re-enabled through the same doors as every other configured record — the admin screens, the CLI, and the configuration
  document — and its changes appear in the same change log.
- **Repositories and forge-backed work sources name their forge.** A repository stops carrying a forge address of its
  own and names a forge instead; so does a work source whose provider is a forge. Each keeps its own credential, so the
  key that reads a tracker's issues and the key that lands into a repository can differ, as they can today. A forge that
  an enabled record still names cannot be retired.
- **One landing seam.** The hub speaks to a forge through one seam that reads a branch's tip, compares two revisions,
  opens, reads, updates, and merges a change, reads and re-runs the checks on a revision, and confirms that a commit
  exists — cut into narrow roles along the lines its callers use, so the garden's check that a delivered commit exists
  depends on reading commits and nothing more. Each forge kind answers in the seam's own vocabulary, so an action reads
  that a change is behind its base or in conflict, never GitHub's `mergeable_state`, and every one of GitHub's words has
  a written place in it. GitHub is the one kind this epic builds. The garden's check that a delivered commit exists
  moves onto the same seam; the work-source family's reads, labels, and closes keep their own seam and only learn their
  address from the forge.
- **A forge's refusals sorted by who can act on them.** Every answer a forge gives is classified once: a forge that is
  briefly unreachable or out of patience is waited out, a race the forge reports is re-read, a conflict goes back to the
  chunk's worker, and a wrong credential or a repository the forge does not know goes to a person, since no worker can
  repair a tenant's configuration. A brief outage is retried within the call before it is waited out at all.
- **Capabilities an action can count on.** Each forge kind declares the capabilities it offers, and each of the hub's
  actions the ones it cannot do without. A landing into a repository whose forge lacks one is refused before the action
  runs, naming the capability and the forge, and brought to a person rather than a worker. A capability an action can
  live without is not required; the action reads what the forge offers and takes its own fallback, as `land` can carry
  on without re-running a check.
- **`land`, ported.** The `land` action is rewritten against the seam and behaves as it does today, so the fleet lands
  as it did before, save that a refusal only a person can resolve now reaches one; the repository's credential is
  revealed inside the forge's adapter and handled nowhere else.
- **Carried over without a hand on it.** An installation becomes one forge per server its records already name, each
  repository and work source pointed at its own, with nothing reconfigured by hand. Two spellings of one server's
  address are one server, a port is part of the address, and the rare installation whose records could not be carried
  without one of them silently changing is refused before anything moves, naming what to settle.
- **Old fields through a transition, then retired.** For a release, every door still accepts the addresses a repository
  and a work source carried before, resolving each to its forge, so a configuration document or a runbook written before
  the epic keeps working while it is moved over. The hub notes every use and warns whoever made it. Once the uses stop,
  the old fields are retired, and anything still sending them is refused with a message naming what replaced it.

## Left for later

Everything the seam does not yet need waits for the epic that needs it. The rate budget the seam's callers share arrives
with the `policies` epic, sized against the polling landing itself does. GitHub stays the only forge kind through the
`policies` epic; a second kind, GitLab, is an epic of its own, `gitlab`, where a chunk first lands across two kinds of
forge and the seam's vocabulary first has to hold for both — one chunk can already land across servers, since each of
its repositories lands against its own. A capability that varies from one repository to another, such as a merge queue
found on one repository and not its neighbour, arrives with the `trains` epic, and until then a forge's capabilities are
its kind's. Authenticating as an application rather than with a token waits for an operator who needs it.
