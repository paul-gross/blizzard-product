# Procedures contract

The epic is finished when an agent told to cut a release, or to move a deployment onto one or back, finds the procedure,
follows it, and proves the outcome without improvising a step. Two procedures live in `blizzard-context`'s `workflows/`
and are routed from its `workflows/index.md`; the workspace keeps only what is particular to its instance.

## Cutting a release — `bzh:release`

`workflows/release.md` keeps its id, its rule that the tag is the release and no other path publishes one, its pre-tag
local runs (`blizzard:e2e`, `blizzard:crash-sweep`, `blizzard:journey`, `blizzard:image-smoke`,
`blizzard:compose-smoke`), its confirmation of the Release, the wheel, and the anonymous image pull, and its
repair-forward rule for a red run. It changes in three places:

- **A tag lands on a cut commit.** "Bump `pyproject.toml` in a release-prep commit" becomes the cut commit — the
  version, the `Release-Cut:` trailer, and the steps that run in it — as
  [release-lines' train](../../release-lines/spec/train.md) §The cut commit defines it, and the candidate is tagged
  there.
- **A stable tag lands on its candidate's commit.** The procedure confirms the commit (`git rev-list -n1 vX.Y.0-rc.K`
  equals the commit being tagged) before pushing, and otherwise points to release-lines' §Promotion for when and how.
- **The upgrade notes point somewhere.** For a release that crosses a boundary, the hand-written **Upgrade notes**
  section points to the release's section of `docs/upgrade.md`, until `epic:release-engineering/notes` assembles it.

`epic:release-engineering/release-lines`' procedures — the train, patches, majors — each end in `bzh:release` for the
tag itself; this epic writes only what the first cut needs of it, and the first cut follows release-lines' train
procedure if it has landed by then ([versions.md](./versions.md) §The first release).

## Moving a deployment onto a release, or back — `bzh:release-deploy`

A new `workflows/release-deploy.md` choreographs any deployment of blizzard — a compose hub and the runner hosts that
work against it — moving onto a named release or back off one. It points to the facts it acts on: `docs/upgrade.md` for
the hub's pull and recreate, `docs/rollback.md` for the store downgrade, `docs/versioning.md` for the window, the
release's notes for what it asks. The sequence:

1. **Read the range.** Every release between the one deployed and the target is crossed; read each one's **Upgrade
   notes**, and, for blizzard's own instance, what the workspace runbook adds. A boundary that orders the halves
   differently from step 2 overrides it, as `docs/upgrade.md`'s sections already do.
2. **Order.** Forward: the hub first, then each runner host, so no runner ever runs ahead of its hub. Back: each runner
   host first, then the hub, for the same reason.
3. **Move the hub.** Pin its exact version and digest, never a moving tag, and let the deployment apply it. Going back,
   the store is downgraded on the outgoing image before the swap. Proof: `/api/health`'s `version` equals the target in
   PEP 440 normal form, and `/api/ready` reports `ready` with the store at the expected revision.
4. **Move each runner host.** Install the target release's wheel from its GitHub Release; going back, downgrade the host
   store on the current install first. Migrate, check the directory (`blizzard runner host --check`), mint every graph
   the range changed, restart. Proof: the host's `/api/health` version equals the target exactly, and each runner ticks
   against the hub.
5. **Record the pins** wherever the deployment keeps them, in a reviewed change.

A proof that fails stops the procedure where it is. The half already moved stays where it is if it is inside the window
with the other; otherwise it is moved back by the same procedure.

## The workspace keeps the instance

`context/project/local-instance.md` §Redeploy + restart runbook becomes the instance's binding of `bzh:release-deploy`:
where each pin lives ([hub-pin.md](./hub-pin.md), [runner-pin.md](./runner-pin.md)), the hub's move as merging the pin
change and running `blizzard-infra`'s `scripts/deploy.sh` with the operator's `blizzard-prod-admin` session, and the
runner host's commands with this machine's paths and unit. It restates none of the sequence. The table at the top of the
file says the hub is moved by a reviewed pin and the runner host after it, by hand, onto each release.

The hub's move is the one step a fleet worker cannot take, since it needs the operator's cloud session; an agent driving
the procedure asks for `aws sso login --profile blizzard-prod-admin` to be run (`! aws sso login …`), and stops if it is
not. The runner host's move keeps the rules `context/project/post-delivery.md` already holds for a fleet worker
restarting the host it runs on.
