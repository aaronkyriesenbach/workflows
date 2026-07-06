# workflows

Shared, reusable GitHub Actions workflows for my private repos.

## docker-build-push.yml

Builds a Docker image, tags it based on the `version` field in a `package.json`
(release tag + `latest` on pushes to `master`, `-SNAPSHOT-<sha>` otherwise),
and pushes to GHCR. Refuses to build a branch push if a release image already
exists for the current version (bump the version first).

### Usage

```yaml
name: Build and Push Docker Image
on:
  push:
    branches:
      - "**"

jobs:
  build-and-push:
    uses: aaronkyriesenbach/workflows/.github/workflows/docker-build-push.yml@master
    with:
      package-json-path: package.json # optional, defaults to package.json
    permissions:
      contents: read
      packages: write
```
