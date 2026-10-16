# Pin change contract

Paths are relative to the `blizzard-infra` repository.

## The workflow

A new workflow, `.github/workflows/release-pin.yml`, opens the change that moves `deploy/release.env`
([hub-pin.md](./hub-pin.md)) onto a newer release. `blizzard` is public and carries no knowledge of where it is
deployed, so nothing in `blizzard` notifies `blizzard-infra`; the private repository watches the public one.

| Trigger             | What it does                                                                                                    |
| ------------------- | --------------------------------------------------------------------------------------------------------------- |
| `schedule`, `*/30`  | Moves the pin forward to the newest published release, if it is newer than the pin                              |
| `workflow_dispatch` | Same, or, given an exact `version` input, writes the pin for that release — older ones included, for a rollback |

## Choosing the release

The workflow lists `paul-gross/blizzard`'s GitHub Releases (`gh api repos/paul-gross/blizzard/releases`), skipping
drafts, and orders them by semver precedence, candidates included: `0.2.0-rc.1` < `0.2.0-rc.2` < `0.2.0` < `0.3.0-rc.1`.
The newest wins. A scheduled run never proposes a release older than the pin, nor one equal to it; only a dispatch with
an explicit `version` writes an older one. A release is eligible only once its image is published, which `release.yml`
guarantees by pushing the image before it creates the Release.

The digest is resolved anonymously from the registry —
`docker buildx imagetools inspect ghcr.io/paul-gross/blizzard-hub:<version> --format '{{.Manifest.Digest}}'`, the
multi-arch index — so the pin names what any host will pull. A version whose tag does not resolve fails the run without
opening anything.

The choice and the rendering are a script, `scripts/release-pin.sh` — given the current pin and the release list, print
the release to pin and the file's new contents, or nothing — unit-tested under `tests/` the way `blizzard`'s
`scripts/image-tags.sh` is, so the precedence rule is tested rather than buried in workflow YAML.
`blizzard-infra:ci-workflows` lints the workflow.

## The change it opens

One pull request against `master`, on the branch `release-pin/<version>`, changing `deploy/release.env` alone, with the
subject `chore(deploy): pin the hub to <version>`. Its description carries:

- the release's URL and whether it is a candidate;
- the release's **Upgrade notes** section, copied from the Release body, or a statement that it has none;
- the range of releases the move crosses, from the pin's version to the new one, since every one of their notes applies;
- what merging does not do: the pin reaches the host only through `scripts/deploy.sh`
  ([procedures.md](./procedures.md)), and the runner host is owed its own move after the hub serves the release.

There is at most one open pin change. A run that finds an open `release-pin/*` change for an older release closes it,
naming the release that superseded it, before it opens its own; a run whose release already has an open change updates
nothing. The workflow never merges.

## Its token

A pull request opened with the workflow's `GITHUB_TOKEN` starts no workflow, so `pr.yml`'s gate would never run on it.
The workflow opens the change with a fine-grained personal access token, the Actions secret `RELEASE_PIN_TOKEN`, scoped
to `paul-gross/blizzard-infra` alone with `contents: write` and `pull-requests: write`. Reading `blizzard`'s releases
needs nothing, since the repository is public. No AWS credential enters any workflow; `context/verification.md` §CI
keeps saying so. `docs/secrets.md` gains the token: where it lives, its scope, and how to rotate it.

The token is created by hand, as an Actions secret of `blizzard-infra`, before the workflow lands: without it every
scheduled run fails.
