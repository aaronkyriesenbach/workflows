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

## 2. Require the `checks` status check before merging (optional)

Only relevant for repos that define a `checks` job (see the README). This
stops the Release PR's merge button from going green while `checks` is
failing.

**Requires GitHub Pro on private repos** — GitHub gates both classic branch
protection and the newer Rulesets API behind Pro for private repos on free
accounts (public repos work on the free plan). The API call below returns
`403 Upgrade to GitHub Pro or make this repository public` on a private repo
without Pro.

Note this is a nice-to-have, not a hard requirement: `docker` already
depends on `needs.checks.result == 'success'` directly, so no broken build
can be published even if the Release PR itself gets merged while red — the
next push (the merge itself) re-runs checks and still blocks `docker` if
they fail. Skipping this step just means the merge button won't warn you
first.

**Via the UI:** repo → Settings → Branches → Add branch protection rule for
`master` → check "Require status checks to pass before merging" → select
the `checks` job's check name (must have run at least once already, or it
won't appear in the list) → Save. Leave "Require a pull request before
merging" unchecked — that would block the direct-to-master pushes these
repos otherwise still support.

**Via `gh`** (public repo, or private with Pro):

```sh
gh api -X PUT "repos/<owner>/<repo>/branches/master/protection" --input - <<'EOF'
{
  "required_status_checks": { "strict": false, "checks": [{ "context": "<checks job name>" }] },
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null
}
EOF
```

Replace `<checks job name>` with the exact `name:` of the `checks` job in
that repo's `docker.yaml` (e.g. `Check and test`, `Build, vet, and test`,
`Build, test, and lint` — check `repos/<owner>/<repo>/commits/master/check-runs`
via `gh api` if unsure of the exact string).

## 3. Create the `release` and `branch-preview` environments

`docker-build-push.yaml` and `npm-build-push.yaml` both select their
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
