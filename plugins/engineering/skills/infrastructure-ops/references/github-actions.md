# GitHub Actions

Read this when writing or reviewing a workflow, setting up OIDC trust for a cloud role, or auditing every workflow in a repository.

Action references below are written `owner/action@<commit-sha> # <tag>`. The slot is left open on purpose: resolve the SHA from the upstream repository when you write the workflow (procedure below). Copying a SHA out of any document, this one included, is how impostor commits get in.

## Pinning an action

```sh
git ls-remote --tags https://github.com/actions/checkout 'refs/tags/v5*'
```

For an annotated tag the output has two lines, `refs/tags/vX.Y.Z` (the tag object) and `refs/tags/vX.Y.Z^{}` (the commit). Pin the `^{}` commit. Because the SHA came from a tag in the upstream repository, it cannot be a commit that exists only in a fork. Write the tag in a trailing comment so update bots and reviewers can read it, and review the upstream diff when a bot proposes a bump. Workflow linters such as zizmor flag unpinned references, SHAs that match no tag, and impostor commits.

Container images used by jobs (`container:`, `services:`, `uses: docker://...`) get pinned by `@sha256:` digest for the same reason.

## Pull request CI (untrusted code, no secrets)

```yaml
name: ci
on:
  pull_request:

permissions: {}

concurrency:
  group: ci-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<commit-sha> # <tag>
        with:
          persist-credentials: false
      - uses: actions/setup-node@<commit-sha> # <tag>
        with:
          node-version-file: .nvmrc
      - run: npm ci
      - run: npm test
```

Fork pull requests under `pull_request` get no secrets other than a read-only `GITHUB_TOKEN`. Dependabot pull requests also get a read-only token, and the only secrets they see are Dependabot secrets. That is the property you want for any job that executes code from the PR.

## Acting on PR results with privileges

When a PR build must lead to something privileged (a comment, a label, a preview deploy), split it. The unprivileged `pull_request` workflow writes its results to a file and uploads it as an artifact. A second workflow reads that artifact as data:

```yaml
name: ci-report
on:
  workflow_run:
    workflows: [ci]
    types: [completed]

permissions: {}

jobs:
  comment:
    if: github.event.workflow_run.event == 'pull_request'
    runs-on: ubuntu-24.04
    permissions:
      actions: read
      pull-requests: write
    steps:
      - uses: actions/download-artifact@<commit-sha> # <tag>
        with:
          name: ci-report
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ github.token }}
      - name: Post summary
        env:
          GH_TOKEN: ${{ github.token }}
          GH_REPO: ${{ github.repository }}
          HEAD_SHA: ${{ github.event.workflow_run.head_sha }}
        run: |
          pr_number=$(jq -r '.pr_number' report.json)
          case "$pr_number" in
            ''|*[!0-9]*) echo "invalid PR number in artifact" >&2; exit 1 ;;
          esac
          actual_head=$(gh pr view "$pr_number" --json headRefOid --jq .headRefOid)
          if [ "$actual_head" != "$HEAD_SHA" ]; then
            echo "artifact PR number does not match the triggering run" >&2
            exit 1
          fi
          jq -r '.summary' report.json > summary.md
          gh pr comment "$pr_number" --body-file summary.md
```

Everything in the artifact is attacker-controlled. The job never checks out the PR, never runs a script from the artifact, validates the PR number as digits, and checks that the PR's head commit is the commit the triggering run built, so a forged artifact cannot redirect the comment to another PR. No cache is restored or saved here.

## Deploy with OIDC, environment approval and serialization

```yaml
name: deploy
on:
  push:
    branches: [main]

permissions: {}

concurrency:
  group: deploy-production
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      id-token: write
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@<commit-sha> # <tag>
        with:
          persist-credentials: false
      - uses: aws-actions/configure-aws-credentials@<commit-sha> # <tag>
        with:
          role-to-assume: ${{ vars.BUILD_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: aws-actions/amazon-ecr-login@<commit-sha> # <tag>
      - uses: docker/setup-buildx-action@<commit-sha> # <tag>
      - id: push
        uses: docker/build-push-action@<commit-sha> # <tag>
        with:
          push: true
          tags: ${{ vars.IMAGE_REPOSITORY }}:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-24.04
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: aws-actions/configure-aws-credentials@<commit-sha> # <tag>
        with:
          role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - name: Roll out the built digest
        env:
          IMAGE: ${{ vars.IMAGE_REPOSITORY }}@${{ needs.build.outputs.digest }}
          CLUSTER: ${{ vars.EKS_CLUSTER }}
          ROLLOUT_TIMEOUT: ${{ vars.ROLLOUT_TIMEOUT }}
        run: |
          aws eks update-kubeconfig --name "$CLUSTER"
          kubectl set image deployment/api api="$IMAGE"
          kubectl rollout status deployment/api --timeout="$ROLLOUT_TIMEOUT"
```

- `permissions: {}` at the top, and each job lists what it uses. `id-token: write` only lets the job request an OIDC token; it grants nothing else.
- The deploy job references the `production` environment, which has required reviewers, self-review prevented, and deployment limited to `main`. The deploy role's trust policy accepts only that environment's `sub`.
- The build role can push images and nothing else; the deploy role can update the cluster and nothing else.
- The image is deployed by the digest the build produced, so a retag cannot change what ships.
- `${{ github.sha }}` and `${{ vars.* }}` are not attacker-controlled; `vars` are set by repository admins.
- Deploy runs queue instead of cancelling each other. A waiting run is replaced by a newer one, which is the behavior you want for "deploy latest main".

## OIDC trust policy (AWS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:environment:production"
        }
      }
    }
  ]
}
```

Use `StringEquals` on `sub` for deploy roles. `StringLike` with `repo:<org>/<repo>:*` accepts every branch, tag, environment and pull request run in the repository. A policy that checks only `aud` accepts tokens from any repository on GitHub, since every workflow that requests a token for AWS asks for the same `sts.amazonaws.com` audience. When a job references an environment, the token's `sub` is the environment form, not the branch form, so a trust policy written for `ref:refs/heads/main` will reject the environment-scoped job; copy the exact `sub` from a real token or the provider docs when setting this up.

## Expression injection

Before:

```yaml
- name: Announce preview
  run: |
    echo "Preview for ${{ github.event.pull_request.title }} at ${{ github.head_ref }}"
```

After:

```yaml
- name: Announce preview
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
    HEAD_REF: ${{ github.head_ref }}
  run: |
    printf 'Preview for %s at %s\n' "$PR_TITLE" "$HEAD_REF"
```

In `actions/github-script`, read the value from `context.payload` or `process.env` inside the script rather than interpolating `${{ }}` into the script body. Values written to `$GITHUB_OUTPUT` and later used as `${{ steps.x.outputs.y }}` inside `run:` carry the taint with them.

Safe to interpolate: `github.sha`, `github.run_id`, `github.repository`, numeric ids such as `github.event.pull_request.number`, `vars.*`, and outputs you computed from those.

## Auditing a repository's workflows

1. List every trigger. For `pull_request_target`, `workflow_run` and `issue_comment`, find what they check out (`ref:` inputs pointing at `head.sha`, `head.ref` or `refs/pull/*/merge`) and whether anything from the PR is executed afterwards: install scripts, builds, tests, local actions (`uses: ./...`), or scripts from a downloaded artifact.
2. Search `run:` and `script:` blocks for `${{ github.event.`, `${{ github.head_ref`, and `${{ steps.*.outputs.* }}` whose source is untrusted.
3. Search `uses:` for anything not followed by a 40-character hex SHA, and container images without `@sha256:`.
4. Check for a top-level `permissions:` block and per-job grants. Flag `write-all`, missing blocks, and `secrets: inherit`.
5. For each secret, find which jobs can read it and whether those jobs are behind an environment with reviewers.
6. Check `runs-on` for self-hosted labels in a public repository.
7. Check `concurrency` on deploy and `terraform apply` jobs for `cancel-in-progress: true`.
8. Check cloud trust policies for `sub` conditions that use wildcards.
9. Run a workflow linter (zizmor covers template injection, dangerous triggers, unpinned uses, excessive permissions, impostor commits and credential persistence), then confirm each finding by hand: a dangerous trigger that never touches PR content is not a finding.
