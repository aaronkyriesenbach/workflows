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
