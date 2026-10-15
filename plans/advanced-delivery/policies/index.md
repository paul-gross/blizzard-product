---
epic: advanced-delivery/policies
refinement: refined
status: horizon
---

# Plan — `epic:advanced-delivery/policies`

Every repository blizzard lands into gets the same habit: open a pull request, wait for checks, merge it. That is right
for a shop that trusts its fleet to merge, and wrong for the two shops beside it. One wants a person's hand on every
merge, because a pull request is where their review happens; the other has no use for a pull request at all and wants
its main branch fast-forwarded the moment the work is proven. Today either of them has to write a graph of their own to
get it, because the landing policy lives in whichever action the graph's delivery node names, and a graph is shared by
every repository a chunk might land in.

This epic moves the policy to where it belongs. How work lands is a fact about the repository, so the repository says
it, and every graph lands into it the same way. A chunk's graph stops choosing how to land and only says what to do with
the answer. Each project also says the order its repositories land in, so a chunk that touches a library and its
consumer lands the library first. Every policy here lands on GitHub, the one forge kind behind the seam, and asks the
seam only for what any forge could offer, so a second kind can join later without a policy changing.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **A landing policy on every repository.** The tenant's repository record carries its policy, chosen from three:
  - **Fast-forward.** The base branch advances to the chunk's proven revision, with no pull request. A base that has
    moved on is brought in base first, so the base's own history stays its first-parent line.
  - **Merge once checks pass.** A pull request is opened, brought up to its base, and merged by the hub when its checks
    pass — what every repository does today.
  - **Park for a person.** A pull request is opened and left for a person to merge; the hub takes over the moment they
    do.

  A project's link to a repository may carry a default of its own only when one earns its place; until then the
  repository's policy is the only one, and every project that links the repository shares it, as they share its branch.
- **Graphs that land anywhere.** A delivery node submits to its branch's flow and routes on the verdict, never naming a
  policy, so one graph lands into a fast-forwarding repository and a parking one without a change. Every policy answers
  in the same words — landed, conflict, failure, or inherited-failure — and a graph authors an edge for each it cares
  about. Every policy waits in the same two ways, too: pending while checks run, or awaiting a person while a parked
  change is unmerged. Neither is a verdict, so a graph authors no edge for either; the chunk simply stays at its
  delivery node. The `land` action runs the repository's policy, so the shipped graphs' delivery nodes name `land` and
  never a policy.
- **What only a person can settle goes to a person.** A person who closes a parked change without merging it has said
  no, and a base its forge will not let anything but a pull request move is a setting to change; neither is a failure an
  agent could repair, so each escalates to a person rather than routing back into the graph.
- **Capabilities checked when the policy is set.** Each policy needs capabilities of the repository's forge — a pull
  request to park one, advancing a ref to fast-forward — and setting a policy the forge cannot honour is refused then,
  naming what is missing, with the same check repeated when a chunk's repositories resolve at landing. What a forge's
  kind cannot know, such as a base protected so that only a pull request may move it, is found at the first landing and
  goes to a person.
- **Honest about what each policy proves.** Fast-forward lands exactly the revision its checks passed on. Merge once
  checks pass proves the change, and a base that moved after its checks passed is merged in by the forge unproven unless
  the forge itself requires a change to be up to date first. Park lands whatever a person merged. Proving the
  combination whatever the policy is the `trains` epic's.
- **A parked change, watched.** While a change waits for a person, its chunk stays `delivering` and its card links to
  the change. The hub notices the merge by polling, every two minutes at the most often, and takes over from the
  revision the person merged. Waiting on a person is not a poll running long, so a parked chunk is never bounced for it;
  a change left unmerged longer than its repository allows instead goes to a person as an escalation. A parked chunk
  holds its runner's environments while it waits, and that bound is what keeps a forgotten review from holding them
  indefinitely.
- **A landing that costs the forge almost nothing.** A pass that finds nothing new spends nothing: answers that have not
  changed come back free, answers that cannot change are read once, and the shipped graphs poll once a minute. Every
  caller holding a forge key — landings, polling, the status sweep, closing work items — then shares that key's rate
  budget, reading what the forge reports of it, and slows down rather than failing when it runs low, the least urgent
  callers first.
- **Landing across repositories, in the project's order.** Each of a chunk's repositories already lands against its own
  server, owner, and credential; this epic adds a policy to that, so each repository lands through its own forge with
  its own policy. A project orders the repositories it links by dragging them into place, and a chunk lands its
  repositories one at a time in its project's order, so a library ranked ahead of its consumer lands first; repositories
  no one has ranked land after the ranked ones, alphabetically. Two projects linking the same repositories may order
  them differently. Landing across repositories is never one atomic act: the hub lands none of the repositories it lands
  itself until every one is ready, then lands them in order, and a parked repository holds every repository after it
  until a person merges. If one is refused after another has landed, the landed one stands and the chunk reconciles as
  it does today.
- **Carried over without a change in behavior.** Every existing repository is given the policy it is landed by today,
  merge once checks pass, so the fleet lands as before until an operator chooses otherwise.

## Left for later

A second kind of forge is the `gitlab` epic's: GitLab joins GitHub behind the seam this epic grows, with no policy
changing. Joining a forge's own merge queue or merge train is the `trains` epic's, because whether a repository has one
is found for that repository rather than known from its forge. A policy of an operator's own, run where the operator
already trusts its code, waits for an operator who needs one; the three here cover the shops this initiative was written
for. A reviewer asking for changes on a parked change, rather than merging or closing it, is `epic:ci-feedback`'s to
hear; until then the change waits and its park timeout brings it back to a person. A forge's webhook is not this
initiative's: polling notices a person's merge within minutes, and that is soon enough for now.
