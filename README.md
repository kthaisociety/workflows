# workflows

Reusable GitHub Actions workflows for KTHAIS app repos. They implement the build and release half of
the delivery flow in
[kthaisociety/infrastructure: docs/delivery-plan.md](https://github.com/kthaisociety/infrastructure/blob/main/docs/delivery-plan.md);
the step-by-step guide is
[docs/app-delivery.md](https://github.com/kthaisociety/infrastructure/blob/main/docs/app-delivery.md).

| Workflow | Called on | Does |
|---|---|---|
| [`semantic-pr.yml`](.github/workflows/semantic-pr.yml) | every PR | Fails unless the PR title is a Conventional Commit (`feat: …`, `fix(api): …`). Make it a required check. |
| [`build.yml`](.github/workflows/build.yml) | push to `main` | Builds the image and pushes `ghcr.io/<owner>/<repo>:sha-<7-char commit>`. Outputs `image`, `tag`, `digest`. |
| [`release.yml`](.github/workflows/release.yml) | push to `main` | release-please's release PR (version + `CHANGELOG.md`). When that PR is merged: waits for the release commit's build, then tags that same digest `X.Y.Z`. |

Not here yet: asking `kthaisociety/deployments` to deploy (staging from `build`, production from
`release`). That comes with `deploy.yml` there.

## Using them

Each workflow file has its caller in its header comment. In short, an app repo has:

```yaml
# .github/workflows/pr.yml
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
jobs:
  title:
    uses: kthaisociety/workflows/.github/workflows/semantic-pr.yml@<tag>
    permissions:
      pull-requests: read
```

```yaml
# .github/workflows/build.yml
on:
  push:
    branches: [main]
jobs:
  build:
    uses: kthaisociety/workflows/.github/workflows/build.yml@<tag>
    permissions:
      contents: read
      packages: write
```

```yaml
# .github/workflows/release.yml — a separate file: it waits for build.yml's run of the release
# commit, which would never finish if both were jobs of the same run.
on:
  push:
    branches: [main]
jobs:
  release:
    uses: kthaisociety/workflows/.github/workflows/release.yml@<tag>
    permissions:
      contents: read
      packages: write
      actions: read
    secrets:
      release-app-client-id: ${{ secrets.RELEASE_APP_CLIENT_ID }}
      release-app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

plus, at its root, release-please's config. For an app with no language-specific version file:

```json
// release-please-config.json
{ "packages": { ".": { "release-type": "simple" } } }
```

```json
// .release-please-manifest.json
{ ".": "0.1.0" }
```

`release-type: simple` keeps the version in `version.txt` and `CHANGELOG.md`. Language types (`node`,
`go`, `python`, …) also bump that language's own files.

The release App (a GitHub App with `contents`, `pull-requests` and `issues` write) must be installed on
the app repo, and its client ID and private key available to it as `RELEASE_APP_CLIENT_ID` and
`RELEASE_APP_PRIVATE_KEY`.

## Rules for this repo

- **Actions are pinned to commit SHAs**, with the version in a comment. Dependabot opens PRs to update
  them. Before adding an action, use its latest official release.
- **Callers pin a release tag** of this repo (`@v0.1.0`), never `@main`, so a change here never reaches
  an app unannounced.
- PR titles are Conventional Commits, and every workflow passes `actionlint` (both checked by `ci.yml`).
