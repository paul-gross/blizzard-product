# Plan — `epic:hub`, remote slice

The hub stops living where the operator sits. Today it is colocated with its runner: one machine holds the fleet's shared memory, and being away from that machine means being away from the fleet. This slice moves the hub off-machine — a VPS, a cloud box, a home server — and lets every runner in the fleet sign in to it from wherever it runs. It is the load-bearing piece of [`milestone:centralized-hub`](../milestones.md).

## What already holds

The architecture — the [separation slice](./hub-separation.md) — was built for this day and does not change. The runner speaks the same outbound-only protocol whether the hub is across a loopback or across the internet — nothing ever dials into a dev box, which is exactly why moving the hub is a deployment decision rather than a redesign. Runners already sign in with hub-minted tokens (the runner-auth slice, delivered), so the hub can answer to the open network without trusting whoever knocks. And the hub is already *eventually reachable* by contract: a runner that loses it keeps working its held chunks, buffers its events, and flushes when the hub returns.

## What to build

- **The off-machine deployment.** A hub an operator can stand up on a box that is not a runner: the packaging, configuration, and operational story (TLS termination, where the store lives, backup posture) for running `blizzard hub host` as a service somewhere the operator only reaches over the network.
- **A fleet of more than one.** Multiple runners registered against one hub, each with its own workspace binding and its own token — the registry, the queue, and acquisition already arbitrate across runners; this proves it with real second machines rather than a colocated pair.
- **Outage hardening, exercised.** The eventual-reachability contract stops being a design property and becomes a tested one: a hub that disappears mid-fleet and returns finds every runner reconciled, every buffered event flushed exactly once, and no chunk lost or doubled.

## What this slice is not

Human-facing reach is not here. The board's login, roles, and phone-friendly shell are `epic:board`'s remote slice; carrying questions to people who are away is `epic:ask-answer`'s. This slice makes the hub *reachable*; those make it *worth reaching*.

## Open questions

- The deployment shape blizzard blesses first: bare systemd on a home server, a container image, or both from day one.
- Whether the hub terminates TLS itself or documents a reverse-proxy front as the supported posture.
