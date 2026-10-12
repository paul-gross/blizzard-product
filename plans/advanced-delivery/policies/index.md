# Plan — `epic:advanced-delivery`, policies slice

Every repository blizzard lands into gets the same habit: open a pull request, wait for checks, merge it. That is right
for a shop that trusts its fleet to merge, and wrong for the two shops beside it. One wants a person's hand on every
merge, because a pull request is where their review happens; the other has no use for a pull request at all and wants
its main branch fast-forwarded the moment the work is proven. Today either of them has to write a graph of their own to
get it, because the landing policy lives in whichever script the graph's delivery node runs, and a graph is shared by
every repository a chunk might land in.

This slice moves the policy to where it belongs. How work lands is a fact about the repository, so the repository says
it, and every graph lands into it the same way. A chunk's graph stops choosing how to land and only says what to do with
the answer. The same slice lets a chunk land across forges, which is where the seam the `forges` slice built first has
to hold for two kinds of forge rather than one.

## What to build

- **A landing policy on every repository.** The tenant's repository record carries its policy, chosen from three:
  - **Fast-forward.** The base branch advances to the chunk's proven revision, with no pull request.
  - **Merge once checks pass.** A pull request is opened, brought up to its base, and merged by the hub when its checks
    pass — what every repository does today.
  - **Park for a person.** A pull request is opened and left for a person to merge; the hub takes over the moment they
    do.

  A project's link to a repository may carry a default of its own only when one earns its place; until then the
  repository's policy is the only one, and every project that links the repository shares it, as they share its branch.
- **Graphs that land anywhere.** A delivery node submits to its branch's flow and routes on the verdict, never naming a
  policy, so one graph lands into a fast-forwarding repository and a parking one without a change. Every policy answers
  in the same words — landed, conflict, failure, or awaiting a person — and a graph authors an edge for each it cares
  about. The packaged landing scripts become the hub's policies, and the shipped graphs' delivery nodes stop naming a
  script.
- **Capabilities checked when the policy is set.** Each policy needs capabilities of the repository's forge — a pull
  request to park one, advancing a ref to fast-forward — and setting a policy the forge cannot honour is refused then,
  naming what is missing, with the same check repeated when a chunk's repositories resolve at landing.
- **A parked change, watched.** While a change waits for a person, its chunk stays `delivering` and its card links to
  the change. The hub notices the merge by polling, every two minutes at the most often, and takes over from the
  revision the person merged; a change closed without merging is a failure the graph routes. A forge may declare a
  webhook, and when it does, the forge's word of a merge brings that moment forward; a late or missed event costs
  minutes and never a landing.
- **One webhook and one budget per forge.** A forge's record gains the webhook it delivers events through and the secret
  that signs them. One event reaches every record it concerns: a merged change wakes the repository's parked landing,
  and the work source whose item it closes. Every caller of a forge — landings, polling, the status sweep, closing work
  items — shares that forge's rate budget, reading what the forge reports of it, and slows down rather than failing when
  it runs low.
- **GitLab, the second forge kind.** GitLab joins GitHub behind the landing seam: refs, comparisons, merge requests,
  pipelines, and the capabilities it offers. Its merge requests and pipelines answer in the seam's vocabulary, so a
  policy that runs on GitHub runs on GitLab without knowing which it is on.
- **Landing across repositories and forges.** A chunk whose repositories live on different forges, or under different
  owners or credentials, is no longer refused; each repository lands through its own forge with its own policy. A
  repository may declare the repositories it lands after, so a library lands before its consumer. A chunk lands into
  none of its repositories until every one of them is ready; if one is refused after another has landed, the landed one
  stands and the chunk reconciles as it does today.
- **Carried over without a change in behavior.** Every existing repository is given the policy it is landed by today,
  merge once checks pass, so the fleet lands as before until an operator chooses otherwise.

## Left for later

Joining a forge's own merge queue or merge train is the `trains` slice's, because whether a repository has one switched
on is a capability of the repository rather than its forge. A policy of an operator's own, written as a script, waits
for an operator who needs one; the three here cover the shops this epic was written for. GitLab as a source of work
items, rather than a place work lands, is not this epic's.
