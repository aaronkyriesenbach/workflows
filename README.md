# workflows

Shared, reusable GitHub Actions workflows for my private repos.

## release-please.yaml

Wraps [`googleapis/release-please-action`](https://github.com/googleapis/release-please-action).
Maintains a standing Release PR from Conventional Commits on `master`; merging
it bumps the version, updates the changelog, and creates a git tag + GitHub
Release. Outputs `release_created` and `version` for downstream jobs.

### Usage

```yaml
release-please:
  if: github.ref_name == 'master'
  uses: aaronkyriesenbach/workflows/.github/workflows/release-please.yaml@master
  permissions:
    contents: write
    pull-requests: write
```

Expects `release-please-config.json` and `.release-please-manifest.json` at
the repo root (override via the `config-file`/`manifest-file` inputs).

## docker-build-push.yaml

Builds a Docker image and pushes to GHCR. On `master`, tags `<version>` (from
the `version` input) and `latest`. On any other branch, tags
`<branch>-<sha>` — branch builds are for testing only and carry no version
number.

### Usage

```yaml
docker:
  needs: release-please
  if: always() && (github.ref_name != 'master' || needs.release-please.outputs.release_created == 'true')
  uses: aaronkyriesenbach/workflows/.github/workflows/docker-build-push.yaml@master
  with:
    version: ${{ needs.release-please.outputs.version }}
  permissions:
    contents: read
    packages: write
```

`if: always()` is required so the `docker` job isn't auto-skipped on branch
pushes, where `release-please` is itself skipped by its own `if`.

## Adding checks (optional, per-repo)

There's no shared "checks" reusable workflow — test/lint/build steps are too
different per stack (Bun, Go, service-container-backed integration tests,
etc.) to genericize without losing real step structure. Instead, each repo
defines its own `checks` job directly in its `docker.yaml`, and the `docker`
job gates on it:

```yaml
checks:
  name: <describe what this does>
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    # ...repo-specific build/lint/test steps...

docker:
  needs: [checks, release-please]
  if: always() && needs.checks.result == 'success' && (github.ref_name != 'master' || needs.release-please.outputs.release_created == 'true')
  uses: aaronkyriesenbach/workflows/.github/workflows/docker-build-push.yaml@master
  with:
    version: ${{ needs.release-please.outputs.version }}
  permissions:
    contents: read
    packages: write
```

Note the extra `needs.checks.result == 'success'` clause — `always()` only
exists to bypass the "skip if a needed job was skipped" default (from
`release-please` being skipped on branches); it would otherwise also bypass a
genuine `checks` failure, so it must be paired with an explicit success check.

If a repo doesn't want checks, just omit the `checks` job and drop it from
`needs`/`if` — `docker`'s condition reverts to the release-please-only form
shown above.

Because everything triggers on `push: branches: "**"`, `checks` also runs
automatically against release-please's own PR branch
(`release-please--branches--master`), giving you a real pass/fail signal on
the Release PR before merging — no extra `pull_request` trigger needed.

See [SETUP.md](./SETUP.md) for one-time manual repo configuration these
workflows depend on.
