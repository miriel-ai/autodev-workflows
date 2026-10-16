# autodev-workflows

Public mirror of **autodev**'s cross-org-callable reusable GitHub Actions
workflows.

autodev's CI/CD lives in a **private** repo (`miriel-ai/miriel-infra`), and
GitHub only lets a private repo's reusable workflows be called from the **same
org** — there is no setting that grants an arbitrary external org access to a
private repo's reusable workflows. So any reusable workflow that has to run in a
**customer's own CI** is published here, in a public repo, where any org can
`uses:` it.

This repo is the **single source of truth** for the workflows it holds: a public
reusable workflow is callable same-org as well as cross-org, so same-org repos
reference these public copies too and there is only ever one definition to keep
in sync.

## Workflows

### `cleanup-ecr-tags.yml`

On PR close / branch delete (reactive mode) or on a schedule (audit mode),
cancels a branch's in-progress builds and batch-deletes its ECR image tags, so
the deploy-controller's "branch is gone → tear the preview stack down" path
fires. It must run in the **calling repo's** CI: it needs `actions: write`
there (to cancel sibling build runs) plus the caller's ECR credentials, which
is why it can't move server-side.

Add it to the calling repo's own `.github/workflows/cleanup-ecr-tags.yml`:

```yaml
name: Cleanup ECR Branch Tags

on:
  pull_request:
    types: [closed]
  delete:

jobs:
  cleanup:
    permissions:
      contents: read
      id-token: write   # assume the ECR role via OIDC
      actions: write    # cancel in-progress build runs
    uses: miriel-ai/autodev-workflows/.github/workflows/cleanup-ecr-tags.yml@v1
    with:
      images_json: '[{"image":"<org>/<ecr-repo>","target_segment":"dev"}]'
      build_workflow_name: build-and-push-to-ecr.yml
      role_to_assume: ${{ vars.AUTODEV_ECR_ROLE_ARN }}
      # build_server_id: build-server-1   # default; must match the build tag prefix
      # aws_region: us-east-2             # default
```

**Authentication is GitHub OIDC only** — there is no static-key fallback. The
caller passes `role_to_assume` (the ARN of an IAM role whose trust policy lets
this repo's OIDC token mint ECR credentials) and grants the calling job
`id-token: write`. The workflow holds no secrets of its own.

`role_to_assume` can come from a repository **variable** (as an input under
`with:`, above) or from a repository **secret**. GitHub does not expose the
`secrets` context inside a caller's `with:` block, so a repo that keeps the ARN
as a secret passes it through the job's `secrets:` block instead — same name,
different block:

```yaml
    uses: miriel-ai/autodev-workflows/.github/workflows/cleanup-ecr-tags.yml@v1
    with:
      images_json: '[{"image":"<org>/<ecr-repo>","target_segment":"dev"}]'
      build_workflow_name: build-and-push-to-ecr.yml
    secrets:
      role_to_assume: ${{ secrets.AWS_ECR_ROLE_ARN }}
```

Pass one or the other; the secret takes precedence if both are set, and the
run fails up front with a clear error if neither is. (`secrets: inherit` also
works, provided the caller's secret is literally named `role_to_assume`.)

For a daily orphan sweep, add a second `schedule`-triggered job that passes
`mode: audit` (and, once a few dry runs look sane, `dry_run: false`). Audit
mode treats a branch's tags as orphaned when the branch no longer exists —
and also when the branch **does** still exist but every pull request whose
head is that branch is closed or merged, with none open. That second rule
catches branches recreated by a push during or after the merge (common with
delete-branch-on-merge), which would otherwise hold their preview alive
indefinitely. It is guarded by:

* `pr_closed_grace_hours` (default `24`): the most recent PR must have closed
  more than this many hours ago, so a just-merged PR's preview isn't yanked
  mid-review.
* A branch with **no PRs at all** keeps its tags (same as before), and any
  **open PR wins** — a branch reused by a new PR is never reaped. Only PRs
  whose head lives in the calling repo count; a fork's PR that reuses the
  branch name neither keeps nor reaps the tags.
* The repository's **default branch** and any **protected branch** are never
  reaped by this rule, whatever their PRs say. If a PR lookup fails, the
  branch is kept.
* `delete_stale_branches` (default `false`): when such a branch is reaped,
  also delete the branch itself. This needs the calling job to grant
  `contents: write` instead of `contents: read`.

The PR lookup needs the audit job's token to read pull requests, so a
private calling repo should add `pull-requests: read` to the audit job's
`permissions` (without it every lookup fails and live branches are simply
kept, as before):

```yaml
  audit:
    permissions:
      contents: read        # write, if delete_stale_branches: true
      id-token: write
      actions: write        # validated for every job, even in audit mode
      pull-requests: read
    uses: miriel-ai/autodev-workflows/.github/workflows/cleanup-ecr-tags.yml@v1
    with:
      mode: audit
      images_json: '[{"image":"<org>/<ecr-repo>","target_segment":"dev"}]'
      build_workflow_name: build-and-push-to-ecr.yml
      role_to_assume: ${{ vars.AUTODEV_ECR_ROLE_ARN }}
      # dry_run: false              # once a few dry runs look sane
      # pr_closed_grace_hours: 24   # default
```

`dry_run` (default `true`) stays report-only throughout: it lists the tags —
and, with `delete_stale_branches`, the branches — that would be reaped without
touching anything. See the workflow's input descriptions for these and the
legacy-tag knobs.

## Versioning

Pin to the moving major tag:

```yaml
uses: miriel-ai/autodev-workflows/.github/workflows/cleanup-ecr-tags.yml@v1
```

`v1` tracks the latest `v1.x.y` release. Pin a specific `v1.x.y` tag instead if
you need an immutable reference.

## What is *not* here

There is no `comment-deployment-url.yml`. The PR preview comment + GitHub
Deployment record used to be a reusable workflow a caller invoked, but the
deploy-controller now renders that surface (every routed host — dashboard,
`-api`, `-anor`) and posts it **server-side via its GitHub App, for every
org**. There is nothing left for a calling repo to mirror, so the workflow has
been retired. Only workflows that *must* run in the caller's own CI (e.g. the
ECR tag cleanup above) live in this public repo.

## Full model

The complete same-org-vs-cross-org split — and why each workflow lands where it
does — is documented in miriel-infra's `.github/workflows/README.md`
(miriel-internal).
