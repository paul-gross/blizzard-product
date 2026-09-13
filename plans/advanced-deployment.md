# Plan — `epic:advanced-deployment`

A chunk that lands has only finished being written. For an operator whose work has to run somewhere, the night is not
over until the change is live, healthy, and known to be both, and today that last stretch belongs to whatever each
deployment happens to be. One piece of blizzard's own instance is published by CI and pulled onto its host by a timer;
another is rebuilt by hand from a runbook; a third ships through a script. An agent asked to improvise across all of
that is the least trustworthy hand to put on production. This epic gives a project a declared, deterministic way to take
landed work the rest of the way, with an agent's judgement called in only where the project asks for it.

The stance that shapes everything below: **blizzard does not deploy.** Nearly every shop already has a pipeline that
builds, tests, and ships, and it already holds the production credentials, the approval gates, and the audit trail.
Blizzard's job is to use that pipeline as it stands, autonomously: to set it moving or follow it, prove what it
produced, and decide what happens next. It never reimplements CI/CD, and it never needs a cloud credential to do its
work.

## The shape every deployment shares

Kubernetes rollouts, ECS services, GitOps controllers, pull-based updaters, progressive-delivery tools, and a person
with a runbook look nothing alike from the outside. Underneath, each one is a trigger, a run that can be identified, a
run that can be observed, an environment that ends up running some version, a decision to promote or hold, and a way
back. The epic is built on that shape rather than on any one of those tools.

The load-bearing observation is that **proof does not depend on mechanism**. If an environment can say which version it
is running, and a declared smoke check can say whether it is healthy, blizzard can prove a deploy without knowing
whether a cluster, a container service, a timer, or a human performed it. Proof is therefore its own seam, separate from
whatever set the deploy in motion.

## What to build

- **A deployment declaration per repository, per environment, owned by the project.** It names the environment, the
  environments it must follow, the driver that moves it, how to read its live version, the smoke checks that prove it
  healthy, the rollback procedure, and how safe that rollback is.
- **Drivers, each declaring what it can do.** A driver starts or finds a run, reports its state (queued, running,
  awaiting approval, succeeded, failed), hands over the evidence a judging agent reads, and optionally approves,
  promotes, or rolls back natively. A policy that asks for a capability a driver lacks is refused when it is declared,
  not discovered mid-deploy. Built in roughly this order:
  - **Follow the pipeline.** The landing already started it; blizzard finds the run for the landed commit, waits it out,
    then proves the environment. This covers most shops without asking them to change anything, GitHub Actions and
    GitLab CI first.
  - **Dispatch the pipeline.** Blizzard starts a named pipeline with inputs, for promotions and for rollback pipelines.
  - **Probe only.** Blizzard triggers nothing and waits for the environment to report the version: the pull-based
    updater, and any process blizzard cannot see into.
  - **GitOps.** Deploying is a version bump committed to a configuration repository, then a wait for the controller to
    report synced and healthy. That commit is itself a landing, so it travels `epic:advanced-delivery`'s policy.
  - **Script.** A declared deterministic command for the runbooks that have no pipeline, with an explicit statement of
    where it runs: some deploys can only be performed on one particular machine, and the hub never reaches into a
    runner's.
  - Native platform drivers — Kubernetes, ECS, Cloud Run — come last if at all, since most organizations reach those
    platforms through a pipeline and a native driver means blizzard holding cloud credentials.
- **A version probe and smoke checks as the universal proof.** A deploy is done when the environment reports the landed
  version and its checks pass, whichever driver ran.
- **Delivery trains.** Once a chunk lands, the train works out which repositories changed, orders their deployments by
  the dependencies the project declared, and moves each only after the one before it is proven.
- **Judgement where declared.** A deploy that needs more than a deterministic verdict runs a graph on a chosen model:
  one agent reads the evidence and decides healthy, broken, or due for rollback; the declared rollback runs as a
  deterministic step; a second agent confirms the rollback took. An agent deploying by hand remains available, as the
  fallback rather than the method.
- **Approval gates surfaced, never bypassed.** A pipeline's own required reviewers and manual jobs arrive in blizzard as
  asks for the person who holds the decision.
- **Per-environment exclusivity and superseding.** Two deploys to one environment never overlap, while different
  environments proceed side by side; when several landings queue behind a deploy, the newest version goes and the ones
  it contains are recorded as carried rather than deployed one by one.

Most of the engine already exists. Hub-executed command nodes start and follow runs; the pending outcome with its poll
interval and timeout is exactly the wait a pipeline needs; authored choices route healthy, broken, and rollback; markers
make a retried deploy skip what is already live. What is new is the declaration, the drivers and probes, per-environment
exclusivity in place of a single fleet-wide slot, dependency ordering, evidence gathering, and scripts that execute on a
runner.

## Principles the design holds to

- **The pipeline keeps the privilege.** Blizzard needs permission to start a named pipeline and read its status, and
  nothing more; production credentials stay where the organization already audits them.
- **Not every rollback is safe.** A deploy that ran a migration cannot always be undone by swapping the version back, as
  blizzard's own hub updater learned. A declaration states whether rollback is safe, needs a human, or is forbidden in
  favor of rolling forward, so no agent improvises the answer.
- **Deterministic first.** A step a script can decide is never handed to a model; a model is called where the project
  declared judgement, and a human where the pipeline or the rollback safety says so.

## Open questions

- How a live version is tied back to a commit when the environment reports something else — a CI run number, an image
  digest, a release tag.
- How a run is identified with certainty when one commit starts several pipelines, or a pipeline is re-run.
- Where evidence beyond pipeline logs comes from — metrics and log queries against the operator's observability stack —
  and whether that is a plug-in of its own.
- Whether promotion between environments is a declared chain inside one train, or a separate train per environment.
- How a script that must run on a particular machine is routed to a runner that holds the right credentials, without the
  hub ever reaching into that machine.
