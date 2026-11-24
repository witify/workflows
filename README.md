# Witify workflows

Reusable GitHub Actions workflows shared by Witify's projects. A project calls one with a few lines; the logic lives here, in one place.

## Release PR

Opens the release pull request from the integration branch (`dev`) into the default branch (`main` or `master`), and refreshes its body on every push. The body holds:

- the **proposed version**: the model chooses `minor` or `major` from the same changes;
- a **changelog** an OpenAI model writes from the pull requests merged since the last release (titles, descriptions, labels) and the commit subjects;
- **deployment notes** read from the diff: added migrations, `.env.example` keys, changed `composer.json` and `package.json`.

It asks nothing of the project: no dependency, no commit convention, no file to keep up to date. Nothing is written to the repository, so there is no version commit to merge back into `dev`.

### Setup

Add `.github/workflows/release-pr.yml` to the project:

```yaml
name: Release PR

on:
  push:
    branches: [dev]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

jobs:
  release-pr:
    uses: witify/workflows/.github/workflows/release-pr.yml@v1
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

The repository needs two things:

- the organization secret `OPENAI_API_KEY` shared with it (organization settings, Secrets and variables, Actions);
- "Allow GitHub Actions to create and approve pull requests" turned on (repository settings, Actions, General).

### Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `head` | `dev` | Branch that holds the work to release. Must match the `branches` of the `push` trigger. |
| `base` | the default branch | Branch that receives the release. |
| `title` | `Release` | Title given to the pull request when it is created. Never changed afterwards. |
| `language` | `fr` | Language of the changelog: `fr` or `en`. |
| `model` | `gpt-6-luna` | OpenAI model that writes the changelog and chooses the bump. |
| `dry_run` | `false` | Write the body to the run summary and create or change nothing. |

### What to know

- **Manual notes are kept.** The generated block sits between `<!-- release-pr:start -->` and `<!-- release-pr:end -->`. Text written outside the markers survives every refresh.
- **It never blocks a release.** Without `OPENAI_API_KEY`, or when the API fails, the pull request lists the merged pull requests as they are, the version is `minor` and the run shows a warning.
- **Merge the release pull request with a merge commit.** A squash leaves the released commits ahead of the default branch, and they come back in the next changelog.
- **CI does not run on the release pull request.** GitHub starts no workflow for a pull request the `GITHUB_TOKEN` opened. A project whose default branch requires checks needs a GitHub App token instead.
- **Try it first** on a project with `dry_run: true`.

## Release

Creates the GitHub release, and its tag, when the release pull request is merged into the default branch. The notes are the changelog and the deployment notes of the pull request.

### Setup

Add `.github/workflows/release.yml` to the project, next to `release-pr.yml`. Both files must be on `dev` and on the default branch.

```yaml
name: Release

on:
  pull_request:
    types: [closed]

permissions:
  contents: write

jobs:
  release:
    uses: witify/workflows/.github/workflows/release.yml@v1
```

### Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `head` | `dev` | Branch that holds the work to release. Same value as in `release-pr.yml`. |
| `base` | the default branch | Branch that receives the release. |
| `dry_run` | `false` | Write the release to the run summary and create nothing. |

### What to know

- **The version** is the latest `vX.Y.Z` tag of the repository, raised by the bump of the pull request: `minor` or `major`, never `patch`. Without a readable bump it is `minor`.
- **A repository without any version tag starts at `v1.0.0`.** To start elsewhere, push the tag of the version already in production before the first release.
- **Only the release pull request makes a release.** A hotfix merged straight into the default branch makes none; its changes are part of the next one.
- **A second run on the same merge makes no second release.**

## Versions

Projects call a major tag (`@v1`), never `@main`. A change merged to `main` reaches no project until the tag moves:

```sh
git tag v1.1.0 && git tag -f v1 && git push origin v1.1.0 && git push -f origin v1
```

Keep the inputs backward compatible within a major version. Before moving the tag, call the workflow at the commit to release from one project, with `dry_run: true`.
