# Terraform

Read this before refactoring Terraform addresses, changing a stateful resource, setting up a backend, or wiring plan and apply into CI.

## Backend

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    key          = "production/data/terraform.tfstate"
    use_lockfile = true
    encrypt      = true
  }
}
```

Bucket, region and KMS key come from a partial configuration file per environment (`terraform init -backend-config=production.s3.tfbackend`); credentials come from the environment or OIDC, never from `-backend-config`, because backend config values are written into `.terraform/` and into plan files.

The state bucket needs versioning (to recover from a bad write or deletion), encryption, public access blocked, and a policy that lets only the plan and apply roles for that stack read it. `use_lockfile` needs `s3:GetObject`, `s3:PutObject` and `s3:DeleteObject` on the `.tflock` object next to the state. DynamoDB-based locking still works but is deprecated.

The 1.11 floor covers `use_lockfile` and write-only arguments; set it to the oldest version you have actually tested.

## Keeping secrets out of state

`sensitive = true` hides a value from CLI output only; state and plan files still contain it. In order of preference:

1. Let the provider or cloud own the secret. `aws_db_instance` with `manage_master_user_password = true` has RDS generate and store the master password in Secrets Manager, so Terraform never sees it.
2. Ephemeral resources and ephemeral variables (Terraform 1.10+) for values needed only during the run, and write-only arguments (`*_wo` with a matching `*_wo_version`, Terraform 1.11+) on resources whose provider supports them. Bump the version argument to push a new value.
3. Create the secret outside Terraform and pass only its identifier (ARN, path) into the configuration.

`random_password.x.result` fed into a resource argument puts the password in state twice. Any existing state that ever held a secret should be treated as having leaked it to everyone who could read that state; rotate after you fix access.

## Refactoring without destroying

Renames, moves into modules and `count` to `for_each` conversions change addresses. Declare each move:

```hcl
moved {
  from = aws_db_instance.main
  to   = module.database.aws_db_instance.this
}

moved {
  from = aws_s3_bucket.uploads[0]
  to   = aws_s3_bucket.uploads["primary"]
}
```

Plan after adding the blocks: the resources must show as moved with no destroy. In a module other configurations consume, keep `moved` blocks permanently, because a caller still on the old address would otherwise plan a delete. Chain moves (`a` to `b`, then `b` to `c`) rather than rewriting old blocks. A `moved` block cannot turn a managed resource into a data source.

To stop managing something without deleting it (handing it to another stack or tool):

```hcl
removed {
  from = aws_s3_bucket.legacy_exports

  lifecycle {
    destroy = false
  }
}
```

## Guarding stateful resources

```hcl
resource "aws_db_instance" "this" {
  identifier                  = var.identifier
  engine                      = "postgres"
  engine_version              = var.engine_version
  instance_class              = var.instance_class
  allocated_storage           = var.allocated_storage_gb
  storage_encrypted           = true
  kms_key_id                  = var.kms_key_arn
  username                    = var.master_username
  manage_master_user_password = true
  backup_retention_period     = var.backup_retention_days
  deletion_protection         = true
  skip_final_snapshot         = false
  final_snapshot_identifier   = "${var.identifier}-final"
  delete_automated_backups    = false

  lifecycle {
    prevent_destroy = true
  }
}
```

- `prevent_destroy` fails any plan that would destroy or replace the resource while this block exists. It does nothing once the block is deleted, and `lifecycle` accepts only literal values, so it cannot be toggled per environment with a variable.
- `deletion_protection` is enforced by AWS, so it also stops a console or CLI delete and survives the Terraform block being removed. Its default is false.
- `delete_automated_backups` defaults to true: without the override, deleting the instance deletes its automated backups with it.
- The final snapshot lives in the same account. Cross-account backup copies are still needed (see deploys-and-observability.md).

Deleting a protected resource on purpose becomes a two-step change: one reviewed apply that turns protection off, then one that deletes. That friction is the point.

## Reviewing a plan

Read the summary line and every resource marked `must be replaced`, `will be destroyed`, or annotated `# forces replacement`. For each, answer: is this expected, does the resource hold data or identity (databases, buckets, keys, DNS zones, IAM roles referenced by ARN elsewhere, load balancers whose address clients use), and what happens between the destroy and the create.

Machine check in CI, run against the saved plan:

```sh
#!/usr/bin/env bash
set -euo pipefail
: "${PLAN_FILE:?path to the saved plan from terraform plan -out}"
: "${PROTECTED_TYPES:?JSON array of resource types that must never be deleted, e.g. [\"aws_db_instance\"]}"

deletes=$(terraform show -json "$PLAN_FILE" | jq -r --argjson protected "$PROTECTED_TYPES" '
  (.resource_changes // [])[]
  | select(any(.change.actions[]; . == "delete"))
  | select(.type | IN($protected[]))
  | "\(.change.actions | join("+")) \(.address) \(.action_reason // "")"')

if [[ -n "$deletes" ]]; then
  printf 'plan deletes or replaces protected resources:\n%s\n' "$deletes" >&2
  exit 1
fi
```

Replacements appear as `delete+create` or `create+delete`, so this catches both. `action_reason` tells you why (`replace_because_cannot_update`, `delete_because_no_resource_config`, `delete_because_each_key` and others). An intentional deletion is approved by changing `PROTECTED_TYPES` for that run in a reviewed commit, not by skipping the check.

## Plan and apply in CI

- Pull request: `terraform plan -out=tfplan` with a read-only role, post the rendered plan for review, run the protected-types check. The plan file contains secrets; do not upload it as a public artifact.
- Merge: apply exactly what was reviewed. Either keep the reviewed plan file (it goes stale if state changes, and Terraform refuses to apply a stale plan) or re-plan on `main` and require approval of that new plan through a protected environment before `terraform apply tfplan`. Never `apply -auto-approve` a plan nobody saw.
- Serialize applies per stack with a concurrency group and `cancel-in-progress: false`. Killing an apply midway leaves partially applied changes and a held lock.
- Pin providers in `required_providers` with a `~>` constraint on the version you tested, commit `.terraform.lock.hcl`, and pin module sources to a version or commit.

## Drift detection

Scheduled job per stack:

```sh
#!/usr/bin/env bash
set -euo pipefail
: "${LOCK_TIMEOUT:?how long to wait for the state lock, e.g. 5m}"

set +e
terraform plan -detailed-exitcode -input=false -lock-timeout="$LOCK_TIMEOUT" -out=drift.tfplan
status=$?
set -e
case "$status" in
  0) echo "no drift" ;;
  2) terraform show -no-color drift.tfplan > drift.txt; exit 2 ;;
  *) exit "$status" ;;
esac
```

Exit code 2 means the real infrastructure no longer matches the code; route it to the owning team as a ticket, and page only for drift in security-relevant resources (security groups, IAM, bucket policies). `terraform plan -refresh-only` shows what changed outside Terraform without proposing to revert it, which is the right view while deciding whether to adopt a console change into code or revert it.

`ignore_changes` is legitimate for attributes another system owns:

```hcl
resource "aws_ecs_service" "api" {
  name            = "api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = var.api_min_capacity

  lifecycle {
    # Application Auto Scaling owns the running count.
    ignore_changes = [desired_count]
  }
}
```

It is not legitimate as a way to make a drift diff go away.
