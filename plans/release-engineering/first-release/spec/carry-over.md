# Carry-over

The instance follows `master` today: the hosted hub's `.env` names `ghcr.io/paul-gross/blizzard-hub:edge`, the updater
recreates the hub whenever that tag moves, and the runner host's venv holds a wheel built from `projects/blizzard`. The
cutover moves each half onto `v0.2.0-rc.1` once, and from then on neither half runs anything but a release.

## The order of landings

1. **`blizzard`: the release tooling and documents.** The rewrites of `docs/versioning.md`, `docs/upgrade.md`,
   `docs/rollback.md`, and `docs/ci.md` ([versions.md](./versions.md)), and the part of
   `epic:release-engineering/release-lines` that the first cut needs ([versions.md](./versions.md) §What the first cut
   needs of the version tooling), which [its carry-over](../../release-lines/spec/carry-over.md) lands before
   `v0.2.0-rc.1`; its release branches, and the mock's matching branch and tag, follow after the cut, created for `0.2`
   after the fact by that same carry-over. Nothing here changes a running deployment. Beside it, and also before the
   candidate is cut, lands `epic:release-engineering/compat-floor`'s first step — its contract documents and
   `window.toml` with `floor` at `0.2` ([its carry-over](../../compat-floor/spec/carry-over.md), step 1) — so that
   `0.2.0` is a baseline for every surface, not only the OpenAPI and dictionary ones. With it, `blizzard-context`'s
   `bzh:release` and `bzh:release-deploy` ([procedures.md](./procedures.md)).
2. **The candidate.** The cut commit for `0.2.0` merges to `master`, and `v0.2.0-rc.1` is tagged at it by `bzh:release`
   ([versions.md](./versions.md) §The first release). The instance is still on `edge`, which at that moment is the same
   source.
3. **`blizzard-infra`: the pin and the updater.** `deploy/release.env` arrives pinning `0.2.0-rc.1` by digest, never
   anything else: the file is born naming a release. With it the updater's rewrite, the hold, the env-file order, the
   contract and smoke tests ([hub-pin.md](./hub-pin.md)), and `release-pin.yml` with its token
   ([pin-change.md](./pin-change.md)).
4. **The hub's cutover**, by hand, once: over SSM, delete the `BLIZZARD_HUB_IMAGE=…:edge` line from
   `/srv/blizzard/deploy/.env` — left in place it is a hold, and the host would stay on `edge` with the updater saying
   so on every run — then run `scripts/deploy.sh`. The install runs the updater, which pulls the candidate and recreates
   the hub. Where `edge` has not moved past a migration since the candidate, the store is at the candidate's head and
   the move is forward; where it has, the updater takes the back path and downgrades on the outgoing `edge` image, which
   is the back path's first real run, so the cutover is best taken before a migration lands after the candidate. Proof:
   `/api/health` reports `0.2.0rc1`.
5. **The workspace: the runner host's cutover.** The **Release** row, the rewritten runbook, and `post-delivery.md`
   ([runner-pin.md](./runner-pin.md)), committed together; then the runbook installs `v0.2.0-rc.1`'s wheel — forward, or
   by the runbook's way back if the host was redeployed past a migration the candidate lacks. Proof: the host reports
   `0.2.0rc1`.
6. **The stable release.** Once the candidate has carried the fleet's ordinary work, `v0.2.0` is cut at its commit. The
   scheduled `release-pin.yml` run opens the hub's pin change; `bzh:release-deploy` moves the hub and then the runner
   host. Both report `0.2.0`.

Between steps 2 and 4 the hub keeps following `edge` and the runner host keeps following landings, so the runbook in
force until step 5 lands is the old one; nothing about running the fleet changes until each half's cutover.

## What keeps running against `edge`

`push.yml` keeps publishing `edge` and `sha-<full-git-sha>`. Its `wire-compat-deployed` job keeps diffing every commit
since the newest successful push run, and keeps blocking `edge` on an unacknowledged break; after step 4 that baseline
no longer describes any deployment of ours. Retiring it, and comparing against the supported releases instead, is
`epic:release-engineering/compat-floor`'s. Until then, a break between `0.2.0` and the next release is caught by the
merge-base check on every pull request and by the `!` marker, not by any comparison against the release.

## Backing the cutover out

Before step 6, the way back to `edge` is the change from step 3 reverted and deployed with `scripts/deploy.sh`, then a
`BLIZZARD_HUB_IMAGE=…:edge` line put back into `.env` for the reverted updater to track, and the runbook from before
step 5. The store needs nothing, since `edge` is never behind the candidate. After step 6 there is no going back to
`edge`, only back to an earlier release.
