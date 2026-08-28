# workflows

Shared, reusable GitHub Actions workflows for my private repos.

## release-please.yaml

Wraps [`googleapis/release-please-action`](https://github.com/googleapis/release-please-action).
Maintains a standing Release PR from Conventional Commits on `master`; merging
it bumps the version, updates the changelog, and creates a git tag + GitHub
Release. Outputs `release_created` and `version` for downstream jobs, plus
`paths-released` — a passthrough of the underlying action's own
`paths_released` output (a JSON array of the manifest paths released in this
run), useful for monorepo callers that gate per-package jobs on whether
their specific path was released.

### Usage

```yaml
release-please:
  needs: checks
  if: github.ref_name == 'master' && needs.checks.result == 'success'
  uses: aaronkyriesenbach/workflows/.github/workflows/release-please.yaml@master
  permissions:
    contents: write
    pull-requests: write
```

Gating on `needs.checks.result == 'success'` matters specifically for the
release-please PR's *merge commit*: that commit is authored by you (not
`GITHUB_TOKEN`), so `checks` genuinely runs on it. Without this gate,
release-please tags/publishes the release regardless of whether `checks`
passed on that commit — an explicit `if:` on a job replaces the implicit
`success()` that `needs:` would otherwise imply, so this has to be spelled
out, it doesn't come for free from `needs: checks` alone. This is
unrelated to (and doesn't conflict with) never requiring `checks` as a
branch-protection status check on the release-please PR itself — see
[SETUP.md](./SETUP.md#2-dont-require-the-checks-status-check-for-merging)
for why that part stays impossible structurally.

If a repo has no `checks` job, drop the gating back to
`if: github.ref_name == 'master'` with no `needs:`.

Expects `release-please-config.json` and `.release-please-manifest.json` at
the repo root (override via the `config-file`/`manifest-file` inputs).

## docker-build-push.yaml

Builds a Docker image and pushes to GHCR. On `master`, tags `<version>` (from
the `version` input) and `latest`. On any other branch, tags
`<branch>-<sha>` — branch builds are for testing only and carry no version
number. Branch pushes require manual approval via the `branch-preview`
environment (see [bun-build-push.yaml](#bun-build-pushyaml) below and
[SETUP.md](./SETUP.md)).

The `version` input is also passed through to `docker/build-push-action` as a
`VERSION` build-arg, so a Dockerfile can declare `ARG VERSION` to bake the
version into the image at build time. The input defaults to an empty string,
so repos that don't pass it (or don't declare `ARG VERSION`) build exactly as
before.

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

## bun-build-push.yaml

Publishes an npm-registry package using Bun. On `master`, publishes the
version already bumped into `package.json` by release-please, tagged
`latest`. On any other branch, publishes a prerelease snapshot version
(`<version>-<branch>.<sha>`) tagged with the sanitized branch name as its
dist-tag, so it never overwrites `latest` and can be installed directly:
`bun add <package>@<branch-name>`.

Uses [`oven-sh/setup-bun`](https://github.com/oven-sh/setup-bun) and
`bun install --frozen-lockfile`/`bun publish` — repos consuming this workflow
need a committed `bun.lock`, not a `package-lock.json`. Publishing goes
through `bun publish`, which does not yet support `--provenance`
([oven-sh/bun#15601](https://github.com/oven-sh/bun/issues/15601)); published
packages won't carry a supply-chain provenance attestation until Bun adds
that support.

Both `docker-build-push.yaml` and `bun-build-push.yaml` gate their branch
publishes behind a `branch-preview` GitHub Environment that requires manual
approval, with a 1-day `timeout-minutes` so an unapproved run cancels itself
instead of waiting forever. `master` releases run under a separate `release`
environment with no protection rules, so real releases are never blocked on
approval. See [SETUP.md](./SETUP.md) for the one-time environment setup.

### Usage

```yaml
bun-publish:
  needs: [checks, release-please]
  if: >
    always() && needs.checks.result == 'success' &&
    (github.ref_name != 'master' || needs.release-please.outputs.release_created == 'true')
  uses: aaronkyriesenbach/workflows/.github/workflows/bun-build-push.yaml@master
  secrets:
    npm-token: ${{ secrets.NPM_TOKEN }}
  permissions:
    contents: read
    id-token: write
```

Requires an `NPM_TOKEN` secret (npm automation token) and, for scoped
packages, `"publishConfig": { "access": "public" }` in `package.json`.

Accepts a `working-directory` input (default `.`) so a monorepo caller can
point at an individual package; all install/build/publish steps run there.
Single-package callers can omit it entirely.

```yaml
bun-publish:
  uses: aaronkyriesenbach/workflows/.github/workflows/bun-build-push.yaml@master
  with:
    working-directory: packages/my-package
  secrets:
    npm-token: ${{ secrets.NPM_TOKEN }}
```

## eas-build.yaml

Triggers an [EAS Build](https://docs.expo.dev/build/introduction/) (and,
by default, an EAS Submit) for Expo/React Native apps. The actual iOS and
Android builds run on Expo's own cloud infrastructure, not on the GitHub
runner — this job only installs deps and calls `eas build`, so no
`macos-latest` runner (and its ~10x per-minute cost) is ever needed, for
either platform. Credentials are EAS-managed; nothing signing-related is
stored as a repo secret beyond `EXPO_TOKEN`.

`--auto-submit` hands a successful build straight to EAS Submit using the
submit profile with the same name as `profile` (defaults to `production`).
EAS Submit always lands iOS in TestFlight and Android on whichever track
`eas.json`'s submit profile specifies — releasing to real users is a manual
step in App Store Connect / Play Console on both platforms and can't be
automated further, so there's no separate gated "promote" job here.

### Usage

```yaml
eas-build:
  needs: [checks, release-please]
  if: >
    always() && needs.checks.result == 'success' &&
    (needs.release-please.outputs.release_created == 'true' ||
      github.event_name == 'workflow_dispatch')
  uses: aaronkyriesenbach/workflows/.github/workflows/eas-build.yaml@master
  with:
    platform: ios
  secrets:
    expo-token: ${{ secrets.EXPO_TOKEN }}
```

Inputs: `platform` (default `ios`), `profile` (default `production`),
`node-version` (default `22`), `install-command`
(default `yarn install --frozen-lockfile` — override for npm/bun/pnpm repos),
`working-directory` (default `.`, for monorepos), `auto-submit`
(default `true`, set `false` to build without submitting).

Requires the consuming repo to have run `eas init` once (links the project
and writes `extra.eas.projectId` into app config) and to have an `eas.json`
with at least a `production` build profile and a matching `production`
submit profile.

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

Because everything triggers on `push: branches: "**"`, `checks` runs
automatically on every real branch push, including PRs opened by other
contributors. The one exception is release-please's own PR branch
(`release-please--branches--master`) — it's authored by `GITHUB_TOKEN`, and
GitHub's loop-prevention rule blocks `GITHUB_TOKEN`-authored pushes from
triggering new workflow runs, so `checks` never runs there. This is expected
and safe, not a bug to fix — see [SETUP.md](./SETUP.md#2-dont-require-the-checks-status-check-for-merging)
for why, and why `master` deliberately has no required-status-check branch
rule as a result.

## zola-pages.yaml

Builds a [Zola](https://www.getzola.org/) site and deploys it straight to
GitHub Pages using GitHub's Actions-based Pages publishing (no `gh-pages`
branch, no artifact committed to git). Runs as two jobs, `build` then
`deploy`, matching GitHub's own recommended split so the `deploy` job can
carry the `github-pages` deployment environment on its own.

Downloads the pinned Zola release binary directly from GitHub Releases
(`x86_64-unknown-linux-gnu`) rather than depending on a Docker image, since
Actions runners already provide everything else `zola build` needs.

### Usage

```yaml
deploy:
  uses: aaronkyriesenbach/workflows/.github/workflows/zola-pages.yaml@master
  with:
    zola-version: "0.23.4"
  permissions:
    contents: read
    pages: write
    id-token: write
```

Inputs: `zola-version` (required, no leading `v`, keep in sync with the
version pinned for local dev/Docker), `working-directory` (default `.`, for
monorepo callers with the Zola project in a subdirectory).

The consuming repo must have GitHub Pages' build source set to **GitHub
Actions** (Settings → Pages → Build and deployment → Source), and, for a
custom domain, the domain configured in that same settings page — see
[SETUP.md](./SETUP.md#4-enable-github-pages-and-configure-a-custom-domain-for-zola-pagesyaml).

## yamllint.yaml

Lints YAML files with [yamllint](https://yamllint.readthedocs.io/) against
this repo's own [`.yamllint`](./.yamllint) config. Accepts a `path` input
(file or directory, default `.github`).

### Usage

```yaml
yamllint:
  uses: aaronkyriesenbach/workflows/.github/workflows/yamllint.yaml@master
  with:
    path: .github
```

This repo uses it against its own `.github` directory as a self-check CI
(see [`ci.yaml`](./.github/workflows/ci.yaml)) — the shared workflows'
own YAML has to lint clean before it's trusted for other repos to call.

See [SETUP.md](./SETUP.md) for one-time manual repo configuration these
workflows depend on.
