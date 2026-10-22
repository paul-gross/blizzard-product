---
epic: release-engineering/supply-chain
refinement: refined
status: in-progress
---

# Plan — `epic:release-engineering/supply-chain`

An engineer weighing whether to run blizzard in their own shop is being asked to hand it a great deal: a hub holding
their forge credentials and their tracker's keys, and a runner that starts coding agents with write access to their
repositories on a machine they own. Before any of that, they will ask the plainest question in software distribution: is
the thing I am about to run the thing these people built? Today blizzard can only answer *trust us*. A release publishes
a wheel attached to a GitHub release and a hub image on GHCR, and neither carries a signature, a record of how it was
built, or a list of what is inside it. The image installs its dependencies afresh at build time, so even its authors
cannot say which versions it carries without pulling it apart. And the runner, the half of blizzard an operator installs
on their own machine, has no published home of its own: it is a wheel downloaded from a release page by hand, with every
dependency resolved to whatever versions happen to be newest on the day.

The hub is pinned easily: an image tag, or better a digest, and the operator knows exactly what they run. The runner
deserves the same, and so does the stranger who wants to check either one without taking blizzard's word for it.

This epic makes every artifact a release publishes one a stranger can verify, and makes the runner installable at a
version an operator names. Nothing about it changes what blizzard does once it is running; it changes what a person can
know about it before they let it run.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **Signed without a key to lose.** Every hub image a release publishes is signed, and its signature names the one
  workflow, at the one release tag, that built it. The signature is minted from the build's own identity rather than
  from a key someone must guard, so there is no secret whose theft would let a stranger sign as blizzard, and anyone can
  check it with ordinary open tooling.
- **How it was made, and what is in it.** Every artifact — the image, the wheel, and each file attached to the release —
  carries an attested record of the workflow run, the commit, and the source that produced it, so a person can trace a
  binary on their machine back to a line of code. The image and the wheel each carry a software bill of materials
  listing every package inside them, published beside them and attested the same way, so the question *are we exposed to
  this week's vulnerability* is answered by reading a file rather than by excavating an image.
- **Checksums beside every attached file.** A release carries one checksums file covering everything attached to it,
  attested like the rest, so one verification vouches for the whole release page.
- **The dependencies a release was proven with are the ones it ships.** The suite proves blizzard against the exact set
  of dependencies the lockfile records. A release attaches that set, every package pinned and hashed, and both the image
  and an operator's runner install from it, so what an operator runs is what the gate proved rather than whatever
  happened to be newest on the morning of the install.
- **A runner pinned as easily as a hub.** A runner is installed straight from the release that published it — the
  attached wheel and the requirements file beside it, fetched from the release page at an exact version — so pinning a
  runner means naming a release, as pinning a hub means naming an image. One command installs it with every dependency
  at the version that release was proven with, and upgrading is naming the next one. The packaged service unit and its
  paths are unchanged; only the way the bits arrive is.
- **Blizzard's own instance verifies first.** Before the hosted hub runs a new image, it checks the image's signature
  against blizzard's release identity and refuses one that fails, as it already refuses an image that fails readiness.
  The runner host is installed only from a wheel whose attestation checks out. The dogfood fleet is the first verifier
  of every release, as it is the first user of every candidate.
- **Confirming a release is a procedure.** `workspace:/context/project/workflows/release.md` already confirms that a cut
  published its release and its image. It gains the steps that confirm what this epic adds — the signature, the
  attestations, the bills of materials, and the checksums — most of them run by the release workflow itself, so an agent
  confirming a cut reads one verdict rather than improvising checks.

## Left for later

The epic stops at what a stranger needs to trust a release, not at the strictest grade of build isolation the field
defines; moving the build into a reusable workflow that the release only calls, so the attested builder is separated
from the code it builds, waits until an operator outside blizzard asks for it. Publishing to a package index waits for
the project's rebrand. Nothing in this design depends on the project's name or on an index — a release page already
serves everything an install needs — so the rebrand changes names, not mechanisms. A source distribution waits too: the
wheel embeds a compiled board, and an archive that must rebuild it with Node on an operator's machine would publish a
second, unproven way to install blizzard. A runner container image waits for an operator who wants one; the runner runs
coding agents in the operator's own workspaces, and a container in between changes what those agents can reach, which is
`epic:security/worker-security`'s question before it is a packaging one. Reproducible builds — byte-identical artifacts
from the same tag on any machine — are left out: attestation answers *who built this and from what*, which is the
question a stranger actually asks.

Which version a deployment runs, and how the instance's pins move, is `epic:release-engineering/first-release`'s; which
lines receive patches, and so which releases stay worth verifying, is `epic:release-engineering/release-lines`'.
