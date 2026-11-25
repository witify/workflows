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

**Strongly recommended: set the custom property `release_flow` to `true` on the repository** (repository settings, Custom properties). Two organization rulesets target every repository where it is `true`, so a project has nothing else to create. From the command line, with a token that has the `admin:org` scope:

```sh
gh api --method PATCH orgs/witify/properties/values --input - <<'EOF'
{ "repository_names": ["<repo>"], "properties": [{ "property_name": "release_flow", "value": "true" }] }
EOF
```


- **Release : merge commit seulement**: pull requests into the default branch can only be merged with a merge commit; "Squash and merge" and "Rebase and merge" disappear from the merge button. The release pull request must be merged that way (see [What to know](#what-to-know)). Pull requests into `dev` keep every method. The rule also requires a pull request for every change to the default branch; organization and repository admins bypass it, so they can still push a hotfix straight to it.
- **dev : suppression interdite**: nobody can delete `dev`, admins included. The release pull request has `dev` as its head branch, so "Automatically delete head branches", or one click on "Delete branch" after the merge, would otherwise delete the integration branch. Pushes and force-pushes to `dev` are unchanged.

Both live in the organization settings, Repository, Rulesets, and appear in each repository's settings, Rules, Rulesets. The repository's default branch must be the branch that receives releases (`main` or `master`), not `dev`: the first ruleset targets the default branch.

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
- **Merge the release pull request with a merge commit.** A squash or a rebase puts new commits on the default branch, so the commits of `dev` are never part of its history: they stay ahead, come back in the next changelog and deployment notes, and the two branches can conflict later. The release itself is still created. Merging the default branch back into `dev` does not fix it; only resetting `dev` to the default branch does. The `release_flow` property of the [setup](#setup) prevents the mistake.
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

## Deploy branch

Builds the frontend assets in CI and force-pushes the source and the build to a deploy branch: the production branch to `deploy`, `dev` to `deploy-dev`. The server (Forge) deploys the deploy branch and never runs npm.

### Setup

Add `.github/workflows/build-deploy-branch.yml` to the project, on `dev` and on the production branch:

```yaml
name: Build deploy branch

on:
  push:
    branches: [dev, main]
  workflow_run:
    workflows: ["Laravel Pint"]
    types: [completed]
    branches: [dev, main]

permissions:
  contents: write

jobs:
  build:
    uses: witify/workflows/.github/workflows/deploy-branch.yml@v1
```

- Use `master` instead of `main` in both `branches` lists when the production branch is `master`.
- Keep the `workflow_run` trigger only when the project has a "Laravel Pint" workflow (use its exact `name`) that pushes a "Fix styling" commit. A push made with `GITHUB_TOKEN` starts no `push` run, so without it the deploy branch misses that commit.
- Then point the Forge site at `deploy` (production) or `deploy-dev` (staging).

### Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `production-branch` | the default branch | Branch deployed to production. |
| `staging-branch` | `dev` | Branch deployed to staging. Empty means no staging deploy. |
| `production-deploy-branch` | `deploy` | Branch the production build is pushed to. |
| `staging-deploy-branch` | `deploy-dev` | Branch the staging build is pushed to. |
| `build-command` | `npm run build` (`pnpm run build` with pnpm) | Command that builds the assets: `npm run production` for Laravel Mix. |
| `package-manager` | `pnpm` with a `pnpm-lock.yaml`, else `npm` | `npm` or `pnpm`. |
| `pnpm-version` | `10` | pnpm version, when `package.json` has no `packageManager` field. |
| `apt-packages` | none | System packages installed before the dependencies, separated by spaces (`libpng-dev` for native npm packages). |
| `paths` | `public/build public/sw.js public/workbox-*.js` | Built files and folders to commit, separated by spaces. Globs are allowed; paths that do not exist are skipped. Laravel Mix: `public/css public/js public/mix-manifest.json`. |
| `node-version` | `.nvmrc`, else `24` | Node version. |
| `composer` | `false` | Install the composer dependencies before the build, for assets imported from `vendor/` (Ziggy, the Vue pages of the `witify/*` packages). |
| `php-version` | `8.4` | PHP version, when `composer` is true. |
| `php-extensions` | `mbstring, fileinfo, dom, mysql, imagick` | PHP extensions, when `composer` is true. |
| `dry_run` | `false` | Build and list the files that would be committed, but push nothing. |

### What to know

- **Pass `production-branch` while the default branch is not the production branch** (a repository whose default branch is still `dev`). Otherwise the default is right.
- **A push to any other branch builds nothing.** The run says so and succeeds.
- **The build fails when none of the paths exists**, so a wrong `paths` or `build-command` shows as a red run instead of an empty deploy.
- **The deploy branch is force-pushed on every run**: never commit to it by hand.

## Versions

Projects call a major tag (`@v1`), never `@main`. A change merged to `main` reaches no project until the tag moves:

```sh
git tag v1.1.0 && git tag -f v1 && git push origin v1.1.0 && git push -f origin v1
```

Keep the inputs backward compatible within a major version. Before moving the tag, call the workflow at the commit to release from one project, with `dry_run: true`.
