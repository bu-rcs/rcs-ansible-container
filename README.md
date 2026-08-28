# BU RCS Systems Ansible Container Build


## Building and releasing the container

Images are built by GitHub Actions and published to
`ghcr.io/bu-rcs/rcs-ansible-container`.

Two kinds of image get published: **dev images** for open pull requests, and
**release images** for tagged versions. Nothing else is published. Pushes to
`main` and manual runs build the image to verify it still compiles, then stop
without pushing.

### Published tags

| Tag | Produced by | Lifetime |
| --- | --- | --- |
| `pr-<N>` | A pull request from a branch in this repo | Deleted when the PR closes |
| `<major>.<minor>.<patch>` | A `v*.*.*` git tag | Kept indefinitely |
| `<major>.<minor>` | A `v*.*.*` git tag | Moves to the newest matching patch |
| `latest` | A `v*.*.*` git tag | Moves to the newest release |
| `sha-<commit>` | A `v*.*.*` git tag | Kept as long as the release image exists |

There is deliberately no `main`, `edge`, or `dev` tag. To try unreleased
changes, use the dev image from the pull request.

### Dev images (pull requests)

Open a pull request against `main`. The build runs automatically and publishes
`ghcr.io/bu-rcs/rcs-ansible-container:pr-<N>`, where `<N>` is the pull request
number. Every push to the PR branch rebuilds and moves the tag, so it always
reflects the current state of the PR.

The tag is deleted automatically when the pull request is merged or closed. Do
not reference it from anything durable.

### Release images

Releases are cut from `main` using an annotated git tag of the form
`v<major>.<minor>.<patch>`:

```bash
gh release create v1.2.0 --target main --generate-notes --title "v1.2.0"
```

`--target main` resolves the tag server-side against the current tip of
`origin/main`, so a stale local checkout cannot tag the wrong commit. Creating a
release from the web UI (**Releases → Draft a new release → Create new tag on
publish**) does the same thing.

Pushing the tag triggers a build that publishes `1.2.0`, `1.2`, `latest`, and
`sha-<commit>`.

Prefer either of the above over `git tag` followed by `git push origin <tag>`.
Forgetting the second step leaves the tag local only, and `git push --tags`
publishes every stale local tag you have.

#### A release is a rebuild, not a retag

The release build is a separate execution from any earlier build of the same
commit. It runs `apt-get update` and installs afresh, so a release image is not
byte-identical to an image built from the same commit at a different time.

The base image (`python:3.12-slim-trixie`) is a floating tag and the OS packages
are unpinned, so rebuilding an old release tag months later would not reproduce
the original image. **The published image is the artifact of record, not the git
tag.** This is why release images are never deleted.

The commit a published image was built from is recorded in its labels:

```bash
docker inspect --format '{{index .Config.Labels "org.opencontainers.image.revision"}}' \
  ghcr.io/bu-rcs/rcs-ansible-container:1.2.0
```

### What does not get published

| Event | Builds | Publishes |
| --- | --- | --- |
| Pull request from this repo | yes | `pr-<N>` |
| Pull request from a fork | yes | nothing |
| Push or merge to `main` | yes | nothing |
| Manual run (`workflow_dispatch`) | yes | nothing |
| `v*.*.*` git tag | yes | release tags |

Merges to `main` build for validation only. A green check on `main` means the
image still builds; it does not mean anything was published.

### Retention and cleanup

Handled by `.github/workflows/cleanup-images.yaml`.

- **On pull request close** — that PR's `pr-<N>` tag is deleted.
- **Weekly (Mondays)** — images whose only tag is `sha-*` and which are older
  than 30 days are deleted, as are untagged images left behind when a `pr-<N>`
  tag moves.

**Any image carrying a release tag (`latest`, `1.2.0`, `1.2`) is excluded from
every cleanup rule** and is never deleted, including its `sha-` tag.

To preview a cleanup without deleting anything, run the workflow manually from
the Actions tab and leave the **Report only** box checked. The log lists every
version considered and what would happen to it. Deleted version IDs are printed
in the log and can be restored through the GitHub packages REST API if needed.

### Listing what is currently published

```bash
gh api --paginate \
  /orgs/bu-rcs/packages/container/rcs-ansible-container/versions \
  --jq '.[] | "\(.id)  \(.updated_at)  \(.metadata.container.tags // ["<untagged>"] | join(","))"'
```

Requires the `read:packages` scope (`gh auth refresh -s read:packages`).
