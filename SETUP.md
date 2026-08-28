# Setup

One-time, per-repo manual configuration required for these workflows to
work. Nothing here is automated by the workflows themselves — GitHub scopes
both of these to repo administration, not the `GITHUB_TOKEN` permissions a
workflow run can grant itself.

## 1. Allow GitHub Actions to create pull requests

`release-please.yaml` needs this to open/update the Release PR. Without it,
the run fails partway through with:

```
release-please failed: GitHub Actions is not permitted to create or approve pull requests.
```

(Note: by that point release-please has usually already created its branch
and commit — the failure happens on the PR-creation API call specifically.
Re-running the workflow after fixing this picks up cleanly.)

**Via the UI:** repo → Settings → Actions → General → "Workflow permissions"
→ check **"Allow GitHub Actions to create and approve pull requests"** → Save.

**Via `gh`:**

```sh
gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow \
  -f default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true
```

## 2. Don't require the `checks` status check for merging

It's tempting to add a branch protection rule requiring the `checks` job to
pass before merging into `master`, so the Release PR's merge button warns
you if `checks` is red. **This doesn't work, and isn't a config mistake — it's
structural.**

`release-please-action` authenticates with `secrets.GITHUB_TOKEN`, so both
the branch it pushes and the PR it opens are attributed to `GITHUB_TOKEN`.
GitHub's own loop-prevention rule means events authored by `GITHUB_TOKEN`
never trigger a new `push`-triggered workflow run ([docs](https://docs.github.com/en/actions/concepts/security/github_token)),
so `checks` (and everything else in `docker.yaml`/`ci.yaml`) simply never
runs on the Release PR's branch — not slowly, not flakily, *never*. If you
require that check for merging, every single Release PR permanently shows
`mergeStateStatus: BLOCKED` and needs an admin bypass (`gh pr merge --admin`
or the web UI's "merge without waiting for requirements" button) to merge at
all, forever. `release-please-action`'s own README confirms this and
recommends a personal access token as the fix — see the note below on why we
deliberately don't do that.

The good news: you don't need the check on the PR to be safe from a broken
release actually publishing. `docker`/`bun-publish` already independently
require `needs.checks.result == 'success'` against the **merge commit**
itself (which *does* trigger `checks` normally, since the merge is performed
by a real account, not `GITHUB_TOKEN`) before they'll build/publish anything.
A bad build still can't ship even with zero branch protection on `master` at
all — the required check on the PR would only have been a visual nicety, and
it can never actually appear for this specific kind of PR.

If you want *some* protection on `master` (recommended) without the
unsatisfiable check requirement, keep it to the basics:

```sh
gh api -X PUT "repos/<owner>/<repo>/branches/master/protection" --input - <<'EOF'
{
  "required_status_checks": null,
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
EOF
```

This still blocks force-pushes and branch deletion, doesn't require a PAT,
and every Release PR merges normally with a plain `gh pr merge --merge` or
the standard green merge button — no admin bypass, no exceptions list to
maintain.

### Why not just use a PAT / GitHub App token instead?

You can — giving `release-please-action` a real user or GitHub App token
instead of `GITHUB_TOKEN` does make `checks` run normally on its PRs, and is
what the upstream project recommends. We've chosen not to, because it adds a
credential to create and keep alive (a PAT) or a one-time GitHub App to set
up (lower-maintenance, but still extra infrastructure) for a check that, per
the above, is already redundant with `docker`/`bun-publish`'s own gate. If
you'd rather have a real green check on the Release PR before merging — e.g.
because other people also open PRs against this repo and you want a uniform
signal — wiring up a GitHub App token for `release-please.yaml`'s `token`
input is the lower-maintenance of the two options (short-lived tokens,
auto-refreshed, no manual rotation).

## 3. Create the `release` and `branch-preview` environments

`docker-build-push.yaml` and `bun-build-push.yaml` both select their
GitHub Environment at runtime (`release` on `master`, `branch-preview`
anywhere else). Neither environment is created automatically, and
`branch-preview` needs to exist with a required reviewer *before* the first
branch push — if it doesn't exist yet, GitHub runs the job with no
protection at all instead of blocking it.

**Via the UI:** repo → Settings → Environments → New environment → name it
`branch-preview` → under "Deployment protection rules" check **"Required
reviewers"** and add yourself (or whoever should approve) → Save protection
rules. Then create a second environment named `release` and leave it with no
protection rules (it exists purely so `master` runs have a named environment
too — functionally identical to no environment).

**Via `gh`:**

```sh
gh api -X PUT repos/<owner>/<repo>/environments/release
gh api -X PUT repos/<owner>/<repo>/environments/branch-preview \
  -f 'reviewers[][type]=User' -F 'reviewers[][id]=<your-user-id>'
```

(`<your-user-id>` is a numeric GitHub user ID, e.g. from `gh api user -q .id`.)

Both workflows also set `timeout-minutes: 1440` on branch runs, so an
unapproved `branch-preview` deployment cancels itself after 1 day instead of
waiting on an approval indefinitely.

## 4. Enable GitHub Pages and configure a custom domain (for `zola-pages.yaml`)

`zola-pages.yaml` deploys through `actions/deploy-pages`, which requires the
repo's Pages source to be **GitHub Actions**, not a branch. This is a repo
setting, not something the workflow can turn on for itself the first time.

**Via the UI:** repo → Settings → Pages → "Build and deployment" → Source →
**GitHub Actions**.

**Via `gh`:**

```sh
gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow
```

(Use `PATCH` instead of `POST` if the Pages site already exists from a
previous branch-based setup.)

For a custom domain, set it the same way. With Actions-based builds, any
`CNAME` file in the deployed artifact is ignored — the domain lives purely in
this repo setting, not in site content.

**Via the UI:** same Settings → Pages screen → "Custom domain" → enter the
domain → Save. Once GitHub verifies the DNS records (see the consuming
repo's own docs for what to add at the registrar), come back and check
**Enforce HTTPS** — it stays greyed out until DNS resolves correctly, and the
certificate can take up to 24 hours to provision after that.

**Via `gh`:**

```sh
gh api -X PUT repos/<owner>/<repo>/pages -f cname=example.com
# after DNS verifies and a cert is issued:
gh api -X PUT repos/<owner>/<repo>/pages -F https_enforced=true
```
