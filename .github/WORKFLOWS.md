# GitHub workflows

This repository uses GitHub Actions to test the provider, check its quality, build and deploy the documentation site, and publish releases. This page is for contributors who need to know what runs on a pull request and how a release is made.

## Index workflow

The [`.index.yaml`](./workflows/.index.yaml) workflow runs on every pull request, and on pushes to `main` that change the provider's code, its tooling, `docker-compose.yml` or the workflows that test and release it. It runs these jobs:

1. **Changes**: on a pull request, works out whether the provider, its tooling or those workflows changed, and whether any shell script changed. A push or a manual run always runs the whole pipeline.
2. **shellcheck**: lints every shell script when a script changed.
3. **Testing and Trivy**: forgego's reusable `testing.yml` and a Trivy scan, in parallel, when the provider changed.
4. **Ignition**: starts a real Ignition gateway with Docker Compose and runs the acceptance tests against it, after testing and Trivy pass.
5. **Release**: on `main` only, after every previous check passes.
6. **Webpage**: on a pull request, the documentation site's checks and build (see [Documentation](#documentation)).
7. **Auto-merge**: merges a Dependabot pull request through forgego's `merge.yml`, once every job above has passed or been skipped.

A new push to a pull request cancels its previous run. On `main`, the documentation site runs from its own push trigger.

## Quality and testing

The provider's tooling comes from [forgego](https://github.com/apollogeddon/forgego). Each tool is pinned in its own module under `.forgego/`, and `Taskfile.yml` runs them. Run a task with:

```bash
go tool -modfile=.forgego/task/go.mod task <name>
```

| Task | What it does |
| :--- | :--- |
| `hooks` | Installs the git hooks. Run it once per clone. |
| `lint` | Formats the code, then lints it and fixes what it can. |
| `format` | Formats the code. |
| `type` | Runs `go vet` on every package. |
| `test` | Runs the unit tests with the race detector and coverage, against OpenTofu. |
| `test:acc` | Runs the acceptance tests against a running gateway, with OpenTofu. |
| `build` | Builds the provider into `dist/`. |
| `release:snapshot` | Builds every release binary into `dist/` without publishing or signing. |
| `security` | Reports known vulnerabilities in code the provider calls (govulncheck). |
| `sync`, `sync-check` | Refreshes, or checks, the files forgego manages. |

The git hooks format staged Go files, check that forgego's files are current and run shellcheck on staged scripts (when it is installed) before a commit, lint before a push, and check commit messages against Conventional Commits.

forgego's [`testing.yml`](https://github.com/apollogeddon/forgego/blob/main/.github/workflows/testing.yml) runs:

- **Quality**: `go mod tidy -diff`, the golangci-lint format check and lint (forgego's base configuration merged with `.golangci.local.yml` into `.golangci.yml`), a check that forgego's files are current, govulncheck and OSV-Scanner.
- **Unit tests**: `task test`.
- **Build**: `task build`.
- **Patch**: on `main`, upgrades modules that govulncheck reports as vulnerable and commits the fix.

**Trivy** scans the repository's Go modules for known vulnerabilities and fails on high or critical ones.

## Acceptance tests

The [`ignition.yaml`](./workflows/ignition.yaml) workflow tests the provider against a real gateway:

- **Gateway**: starts an Ignition 8.3 gateway from `docker-compose.yml`, which restores the seed backup in `assets/ignition.gwbk`.
- **Readiness**: waits until the gateway responds.
- **Tests**: runs `task test:acc`, which creates, reads, updates and deletes each resource through the live API with OpenTofu.

The unit tests, the acceptance tests and the documentation build all use the OpenTofu release pinned in `.github/scripts/install-tofu.sh`, which verifies the download against the release's checksums.

## Release process

The [`release.yaml`](./workflows/release.yaml) workflow versions and publishes the provider:

- **Version**: calls [forgego](https://github.com/apollogeddon/forgego)'s `version.yml`, whose release-please manages version bumps and `CHANGELOG.md` from Conventional Commits, and requests a review of the release pull request it opens.
- **Publish**: when a release was created, GoReleaser builds the provider in the layout the Terraform Registry expects (`.goreleaser.yaml`): a zip per platform, a `SHA256SUMS` file signed with the release GPG key, and the provider manifest (`terraform-registry-manifest.json`). It stays here rather than in forgego's `service.yml`, which doesn't sign the checksums.
- **Webpage**: rebuilds and deploys the documentation site, so its provider mirror lists the new release.

release-please creates each release as a draft; GoReleaser attaches the files and then publishes it, so the flow also works with immutable releases. Releases are published on GitHub only: the provider is not yet listed on the OpenTofu or Terraform registry. To try the build locally, run `task release:snapshot`, which skips signing.

## Documentation

The [`webpage.yaml`](./workflows/webpage.yaml) workflow builds and deploys the [Astro](https://astro.build/) documentation site in `webpage/`:

- **Quality**: calls forgejs's `quality.yml`: Gitleaks over the whole repository, OSV-Scanner on the site's dependencies, Biome and the type check.
- **Markdown**: lints every Markdown file in the repository with `markdownlint-cli2`.
- **Build**: generates the provider reference with `tfplugindocs` from the schema OpenTofu reports and from `examples/`, copies it into the site with `migrate-docs.sh`, and builds the site. The build also runs on pull requests, so a broken site fails the pull request.
- **Deploy**: on `main`, publishes the site to GitHub Pages.

## Configuration

| File | Purpose |
| :--- | :--- |
| `release.json` | release-please configuration. |
| `.release.json` | release-please manifest: the current version. |
| `dependabot.yml` | Keeps GitHub Actions, Go modules and npm packages up to date, with a 3-day cooldown. |
