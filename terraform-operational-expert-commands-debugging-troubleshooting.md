# Terraform Operational Expert Commands, Debugging, and Troubleshooting with STAR Method

This guide is for day-to-day Terraform operations in production. It focuses on expert commands, safe debugging, troubleshooting flows, and STAR-method use cases.

## How to read this guide

- **Command**: Terraform, shell, or helper command used by operators.
- **When to use**: The operational scenario.
- **Debugging purpose**: What the command helps confirm.
- **Safe production note**: Guardrail before using it in production.
- **STAR use case**: Situation, Task, Action, Result format for interviews, incident reviews, and runbooks.

> Production rule: never run `apply`, `destroy`, `state rm`, `state mv`, or `force-unlock` against production without confirming workspace, backend, cloud account, region, plan output, and approval process.

## Quick production triage flow

1. **Confirm target**
   ```bash
   terraform workspace show
   terraform version
   terraform providers
   ```
2. **Initialize safely**
   ```bash
   terraform init -reconfigure
   terraform validate
   ```
3. **Check formatting and static syntax**
   ```bash
   terraform fmt -check -recursive
   terraform validate
   ```
4. **Generate a plan without changing infrastructure**
   ```bash
   terraform plan -out=tfplan
   terraform show tfplan
   ```
5. **Inspect destructive changes**
   ```bash
   terraform show -json tfplan > tfplan.json
   ```
6. **Check state and drift**
   ```bash
   terraform state list
   terraform plan -refresh-only
   ```
7. **Enable debug logs only when needed**
   ```bash
   TF_LOG=DEBUG TF_LOG_PATH=terraform-debug.log terraform plan
   ```
8. **Apply only an approved saved plan**
   ```bash
   terraform apply tfplan
   ```

## Expert command reference

| # | Command | When to use | Debugging purpose | Safe production note |
|---:|---|---|---|---|
| 1 | `terraform version` | Start of every investigation. | Confirms CLI version and provider compatibility warnings. | Compare with `required_version` before changing anything. |
| 2 | `terraform init` | First run or after provider/module/backend changes. | Downloads providers and modules. | Review backend prompt carefully. |
| 3 | `terraform init -reconfigure` | Backend config changed or workspace moved. | Rebinds current directory to backend settings. | Confirm backend bucket/key/workspace before accepting migration. |
| 4 | `terraform init -migrate-state` | Moving state from one backend to another. | Migrates state through Terraform workflow. | Backup source state first and run during change window. |
| 5 | `terraform init -upgrade` | Planned provider or module upgrade. | Updates dependency selections. | Review `.terraform.lock.hcl` changes in PR. |
| 6 | `terraform validate` | Before plan or PR merge. | Checks Terraform syntax and internal consistency. | Validation does not check all cloud API permissions. |
| 7 | `terraform fmt -check -recursive` | CI quality gate. | Detects formatting drift. | Use `terraform fmt -recursive` locally to fix. |
| 8 | `terraform providers` | Provider troubleshooting. | Shows provider requirements by module. | Useful before deleting provider aliases. |
| 9 | `terraform providers lock` | Multi-platform provider lock generation. | Adds provider checksums for target platforms. | Commit lock file after review. |
| 10 | `terraform get -update` | Module source changed. | Refreshes downloaded modules. | Prefer pinned module versions. |
| 11 | `terraform workspace list` | Multi-workspace repository. | Shows available workspaces. | Do not rely only on workspace for critical environment isolation. |
| 12 | `terraform workspace show` | Before plan/apply/destroy. | Confirms active workspace. | Stop immediately if it shows production unexpectedly. |
| 13 | `terraform workspace select <name>` | Switch environment workspace. | Targets existing workspace state. | Confirm backend and cloud account after switching. |
| 14 | `terraform workspace new <name>` | Create ephemeral or new environment state. | Creates separate state namespace. | Ensure naming policy and cleanup plan exist. |
| 15 | `terraform plan` | Preview changes. | Shows create, update, replace, and destroy actions. | Never treat plan as harmless if it uses live credentials with refresh. |
| 16 | `terraform plan -out=tfplan` | Approval workflow. | Saves exact reviewed plan. | Apply the same file, not a newly generated plan. |
| 17 | `terraform plan -refresh-only` | Drift detection. | Compares state with real infrastructure and updates state only if applied. | Review carefully before applying refresh-only changes. |
| 18 | `terraform plan -destroy` | Decommission review. | Shows everything Terraform would destroy. | Require explicit approval and backup validation. |
| 19 | `terraform plan -target=<address>` | Emergency targeted operation. | Limits graph to selected resource. | Use rarely; it can create incomplete convergence. |
| 20 | `terraform plan -replace=<address>` | Force replacement safely. | Shows impact of recreating a resource. | Prefer this over old `terraform taint`. |
| 21 | `terraform plan -var-file=prod.tfvars` | Environment-specific inputs. | Confirms variables for an environment. | Verify the tfvars file matches target environment. |
| 22 | `terraform show tfplan` | Human-readable saved plan review. | Displays the saved binary plan. | Use before approval. |
| 23 | `terraform show -json tfplan` | Automated policy or diff analysis. | Produces machine-readable plan. | Protect JSON plan because it can contain sensitive values. |
| 24 | `terraform apply tfplan` | Approved deployment. | Applies exactly what was reviewed. | Best practice for production. |
| 25 | `terraform apply -auto-approve` | Automation only. | Skips interactive approval. | Use only in controlled CI with prior approval gates. |
| 26 | `terraform apply -refresh-only` | Accept known drift into state. | Updates state to match remote objects. | Does not change remote infrastructure but can hide drift if misused. |
| 27 | `terraform destroy` | Full environment teardown. | Deletes all managed objects in current state. | Disable or heavily restrict for production. |
| 28 | `terraform destroy -target=<address>` | Controlled resource removal. | Destroys selected resource and dependencies. | Prefer code removal plus plan unless emergency. |
| 29 | `terraform state list` | State inspection. | Lists all managed resource addresses. | Read-only and safe. |
| 30 | `terraform state show <address>` | Inspect a state object. | Displays attributes stored in state. | Output may include secrets. |
| 31 | `terraform state pull` | Backup or inspect remote state. | Downloads current state JSON. | Store securely; state can contain secrets. |
| 32 | `terraform state push <file>` | Disaster recovery state restore. | Uploads local state to backend. | Dangerous; use only with backups, lock, and approvals. |
| 33 | `terraform state mv <old> <new>` | Rename or refactor resources. | Moves state address without changing real resource. | Prefer `moved` blocks for repeatable migrations. |
| 34 | `terraform state rm <address>` | Stop managing a resource. | Removes object from state but leaves cloud resource. | Document new owner or cleanup path. |
| 35 | `terraform import <address> <id>` | Bring existing resource under Terraform. | Maps real resource into state. | Write matching config before import when possible. |
| 36 | `terraform console` | Expression debugging. | Evaluates variables, locals, functions, and expressions. | Does not change infrastructure. |
| 37 | `terraform graph` | Dependency debugging. | Produces dependency graph DOT output. | Useful for circular dependency analysis. |
| 38 | `terraform output` | Read stack outputs. | Shows exported values. | Sensitive outputs are hidden unless explicitly requested. |
| 39 | `terraform output -json` | Automation integration. | Machine-readable outputs. | Protect output files because values may be sensitive. |
| 40 | `terraform refresh` | Legacy state refresh. | Updates state from remote infrastructure. | Prefer `terraform apply -refresh-only` in modern workflows. |
| 41 | `terraform force-unlock <lock-id>` | Stale lock recovery. | Removes a backend lock. | Only after proving no apply is still running. |
| 42 | `terraform taint <address>` | Legacy replacement marking. | Marks resource for recreation. | Prefer `-replace` because it is plan-scoped. |
| 43 | `terraform untaint <address>` | Cancel old taint. | Removes taint marker from state. | Verify why the resource was tainted. |
| 44 | `terraform login` | Terraform Cloud/Enterprise auth. | Stores API token locally. | Avoid on shared machines. |
| 45 | `terraform logout` | Remove Terraform Cloud auth. | Deletes stored credentials. | Useful during credential rotation. |
| 46 | `TF_LOG=TRACE terraform plan` | Deep provider debugging. | Emits detailed Terraform and provider logs. | Logs can contain secrets; store and share carefully. |
| 47 | `TF_LOG=DEBUG terraform apply` | Debug apply failure. | Shows provider requests and internal decisions. | Prefer running against non-production first. |
| 48 | `TF_LOG_PATH=terraform.log terraform plan` | Persist debug logs. | Writes logs to file for analysis. | Add log files to `.gitignore`. |
| 49 | `terraform plan -lock-timeout=10m` | Busy backend lock. | Waits for existing state lock to release. | Better than force-unlock during normal pipeline contention. |
| 50 | `terraform apply -parallelism=5` | API throttling or ordering pressure. | Reduces concurrent provider API calls. | Slower but safer for rate-limited APIs. |
| 51 | `terraform providers schema -json` | Provider schema analysis. | Shows attributes, computed fields, and nesting. | Useful for custom policy tooling. |
| 52 | `terraform metadata functions` | Function discovery in newer Terraform versions. | Lists Terraform language functions. | Availability depends on CLI version. |
| 53 | `terraform test` | Module testing. | Runs native Terraform tests. | Use before publishing shared modules. |
| 54 | `terraform test -filter=<test>` | Targeted module test. | Runs a specific test file or test. | Helpful during module debugging. |
| 55 | `terraform fmt -diff` | Review style changes. | Shows formatting differences. | Safe in PR checks. |
| 56 | `terraform validate -json` | CI validation parsing. | Returns machine-readable validation diagnostics. | Good for automated annotations. |
| 57 | `terraform plan -detailed-exitcode` | CI drift or change detection. | Exit code `0` no diff, `2` diff, `1` error. | CI must handle code `2` as expected. |
| 58 | `terraform apply -lock=false` | Special recovery only. | Disables state lock. | Avoid in production; can corrupt state if concurrent run exists. |
| 59 | `terraform state replace-provider` | Provider source migration. | Rewrites provider references in state. | Backup state first. |
| 60 | `terraform plan -generate-config-out=generated.tf` | Import-block workflow in newer Terraform versions. | Generates starting configuration for resources declared in `import` blocks. | Review generated config before use. |

## STAR troubleshooting playbooks

### 1. Backend initialization failure

**Situation:** `terraform init` fails after a backend bucket, key, workspace, or credentials change.

**Task:** Restore safe access to the correct remote state without accidentally creating a new state file.

**Action / Solution:**

```bash
terraform version
terraform init -reconfigure
terraform workspace show
terraform state list
```

- Confirm backend bucket/container, key/path, region, workspace, and cloud identity.
- Check that backend IAM permissions allow read, write, list, lock, and unlock operations.
- If migrating state, backup first and use `terraform init -migrate-state`.

**Result:** Terraform reconnects to the expected backend and avoids accidental duplicate infrastructure.

**Production use case:** Moving production state from local backend to S3, Terraform Cloud, Azure Storage, or GCS.

---

### 2. State lock is stuck

**Situation:** A pipeline failed and Terraform reports that the state is locked.

**Task:** Recover deployment ability without risking concurrent state writes.

**Action / Solution:**

```bash
terraform plan -lock-timeout=10m
terraform force-unlock <LOCK_ID>
```

- First verify no other CI job or engineer is still running `plan` or `apply`.
- Prefer waiting with `-lock-timeout`.
- Use `force-unlock` only after checking pipeline logs and process ownership.

**Result:** The stale lock is cleared safely and normal Terraform operations resume.

**Production use case:** Failed Jenkins, GitHub Actions, GitLab, or Terraform Cloud run holding a backend lock.

---

### 3. Drift detected in production

**Situation:** `terraform plan` shows changes even though no Terraform code changed.

**Task:** Determine whether drift was caused by manual console changes, autoscaling, provider changes, or external controllers.

**Action / Solution:**

```bash
terraform plan -refresh-only
terraform state show <address>
terraform show -json tfplan > tfplan.json
```

- Compare cloud console values with Terraform configuration.
- Import intentional manual changes into code.
- Use narrow `ignore_changes` only for fields owned by autoscaling or another controller.
- Revert unauthorized manual changes through Terraform.

**Result:** Terraform becomes the source of truth again and future plans become clean.

**Production use case:** Security group hotfixes, manually changed instance sizes, autoscaling desired capacity, or altered tags.

---

### 4. Plan shows unexpected resource replacement

**Situation:** A plan includes `-/+` replacement for a database, load balancer, cluster, or other critical resource.

**Task:** Identify the exact attribute forcing replacement and prevent accidental outage.

**Action / Solution:**

```bash
terraform plan -out=tfplan
terraform show tfplan
terraform show -json tfplan > tfplan.json
terraform state show <address>
```

- Inspect which attribute is marked `forces replacement`.
- Check provider documentation for immutable fields.
- Add `prevent_destroy` for critical resources.
- Use blue-green replacement, snapshots, backups, DNS cutover, or manual migration if replacement is required.

**Result:** Destructive replacement is either avoided or performed through a controlled migration plan.

**Production use case:** RDS instance class family change, subnet group changes, ALB name changes, Kubernetes cluster recreation.

---

### 5. Apply failed halfway

**Situation:** Terraform created some resources but failed before completion.

**Task:** Reconcile Terraform state with real infrastructure before re-running blindly.

**Action / Solution:**

```bash
terraform state list
terraform plan
terraform state show <address>
terraform import <address> <remote-id>
terraform state rm <address>
```

- Run a fresh plan to see Terraform's current view.
- Import resources that exist remotely but are missing from state.
- Remove state entries only when Terraform should no longer manage that resource.
- Avoid manual deletion until you understand dependencies.

**Result:** State and infrastructure converge again without duplicate resources or accidental deletion.

**Production use case:** Network stack partially created before quota, IAM, DNS, or provider timeout failure.

---

### 6. Provider authentication or authorization failure

**Situation:** Terraform fails with access denied, expired token, invalid credentials, or wrong account errors.

**Task:** Confirm identity, permissions, and target account before changing infrastructure.

**Action / Solution:**

```bash
terraform providers
terraform plan
# AWS example
aws sts get-caller-identity
# Azure example
az account show
# GCP example
gcloud auth list
gcloud config get-value project
```

- Validate assumed role, subscription, project, region, and provider aliases.
- Prefer short-lived OIDC or workload identity over static access keys.
- Add account ID or project ID validation in Terraform variables or data sources.

**Result:** Terraform runs with the intended identity and avoids cross-account deployment mistakes.

**Production use case:** CI deployment role changed, token expired, or provider alias pointed to the wrong account.

---

### 7. API throttling or timeout during apply

**Situation:** Apply fails intermittently with rate limit, throttling, timeout, or too many requests errors.

**Task:** Reduce provider API pressure and make applies reliable.

**Action / Solution:**

```bash
terraform apply -parallelism=5 tfplan
terraform plan -parallelism=5
```

- Lower `-parallelism` for large changes.
- Split very large root modules into smaller stacks.
- Stagger CI jobs that target the same cloud service.
- Check provider retry settings and cloud service quotas.

**Result:** Apply success rate improves and cloud APIs are not overloaded.

**Production use case:** Creating many IAM policies, DNS records, Kubernetes objects, or networking resources.

---

### 8. Dependency cycle error

**Situation:** Terraform cannot build a graph and reports a cycle between resources or modules.

**Task:** Break the circular dependency while keeping resource ownership clear.

**Action / Solution:**

```bash
terraform graph > graph.dot
terraform validate
terraform plan
```

- Identify resources that depend on each other's computed attributes.
- Replace unnecessary `depends_on` with direct references.
- Split infrastructure into layers, such as network first and application second.
- Pass outputs between stacks instead of bidirectional references.

**Result:** Terraform can calculate a valid order of operations.

**Production use case:** Security groups referencing each other, Kubernetes provider depending on cluster resources, VPC endpoints and route dependencies.

---

### 9. `count` index caused wrong resource change

**Situation:** Removing one list item causes Terraform to update or destroy unrelated resources due to shifted indexes.

**Task:** Preserve identity for repeated resources.

**Action / Solution:**

```hcl
# Safer pattern
resource "aws_security_group_rule" "ingress" {
  for_each = var.ingress_rules
  # each.key gives stable identity
}
```

```bash
terraform state mv 'aws_instance.web[0]' 'aws_instance.web["app-a"]'
```

- Convert list-based `count` to map-based `for_each` with stable keys.
- Use `moved` blocks or `terraform state mv` during migration.

**Result:** Only the intended resource changes during list edits.

**Production use case:** Subnets, DNS records, firewall rules, IAM users, and queue definitions.

---

### 10. Importing existing production resource

**Situation:** A manually created resource must become managed by Terraform.

**Task:** Import without changing or replacing the live resource.

**Action / Solution:**

```bash
terraform import <resource_address> <provider_resource_id>
terraform state show <resource_address>
terraform plan
```

- Write configuration that matches the live resource before import.
- Run plan until Terraform shows no unexpected changes.
- Use generated import config where supported, then clean and review it.

**Result:** Existing infrastructure becomes managed safely by Terraform.

**Production use case:** Bringing manually created S3 buckets, IAM roles, DNS zones, VPCs, or databases under IaC.

---

### 11. Secret appears in state or plan output

**Situation:** Sensitive values show in state, plan JSON, debug logs, or CI artifacts.

**Task:** Reduce exposure and rotate compromised credentials if necessary.

**Action / Solution:**

```bash
terraform state show <address>
terraform output
terraform output -json
```

- Mark outputs as `sensitive = true`.
- Store passwords and tokens in secret managers instead of plain variables.
- Restrict backend and CI artifact access.
- Rotate leaked secrets and remove logs from shared storage.

**Result:** Secrets are protected and future Terraform runs avoid exposing them casually.

**Production use case:** Database passwords, API tokens, private keys, Kubernetes credentials, and service account keys.

---

### 12. Module upgrade creates unexpected changes

**Situation:** Updating a module version changes many resources or plans deletion.

**Task:** Understand module changes before applying to production.

**Action / Solution:**

```bash
terraform init -upgrade
terraform plan -out=tfplan
terraform show tfplan
```

- Read module changelog and migration guide.
- Upgrade first in dev or staging.
- Pin module versions and avoid branch-based module sources.
- Use `moved` blocks for renamed resources.

**Result:** Module upgrades become controlled and reversible.

**Production use case:** Upgrading shared VPC, ECS, EKS, RDS, IAM, or monitoring modules.

---

### 13. Provider upgrade changes plan output

**Situation:** `.terraform.lock.hcl` changes and plans show different behavior after provider upgrade.

**Task:** Verify that provider changes are expected and safe.

**Action / Solution:**

```bash
terraform init -upgrade
terraform providers lock
terraform plan
```

- Review provider release notes.
- Test in lower environments.
- Keep lock file committed.
- Roll back provider version if the diff is unsafe.

**Result:** Provider upgrades are deliberate instead of accidental production changes.

**Production use case:** AWS provider, AzureRM provider, Google provider, Kubernetes provider, and Helm provider upgrades.

---

### 14. Wrong workspace or environment selected

**Situation:** A user is about to run Terraform against production while intending to use staging.

**Task:** Stop accidental changes before plan or apply.

**Action / Solution:**

```bash
terraform workspace show
terraform plan -var-file=staging.tfvars
```

- Add environment assertions in variables or data sources.
- Use separate root modules or pipelines for high-risk environments.
- Display active account, region, and workspace in CI logs.

**Result:** Environment targeting mistakes are caught early.

**Production use case:** Shared repository with `dev`, `stage`, and `prod` workspaces.

---

### 15. Destroy command risk

**Situation:** An operator runs or schedules a destroy command against the wrong state.

**Task:** Prevent accidental full-environment deletion.

**Action / Solution:**

```bash
terraform plan -destroy
terraform destroy
```

- Restrict who can run destroy in CI.
- Use `prevent_destroy` for critical resources.
- Require manual approvals and typed environment confirmation.
- Verify backups, snapshots, and dependency impact.

**Result:** Destructive operations require deliberate review and cannot happen casually.

**Production use case:** Production account, shared networking, databases, state buckets, and DNS zones.

---

## Debug logging levels

| Log level | Command example | Use case | Warning |
|---|---|---|---|
| `ERROR` | `TF_LOG=ERROR terraform plan` | Show only serious errors. | Minimal diagnostic context. |
| `WARN` | `TF_LOG=WARN terraform plan` | Capture warnings during CI. | May miss provider request details. |
| `INFO` | `TF_LOG=INFO terraform plan` | Understand high-level workflow. | Can still be noisy. |
| `DEBUG` | `TF_LOG=DEBUG TF_LOG_PATH=terraform-debug.log terraform plan` | Provider and Terraform debugging. | Logs may contain secrets. |
| `TRACE` | `TF_LOG=TRACE TF_LOG_PATH=terraform-trace.log terraform plan` | Deep internal troubleshooting. | Very verbose and sensitive. |

## Common error messages and fixes

| Error message | Likely cause | Expert troubleshooting commands | Solution |
|---|---|---|---|
| `Error acquiring the state lock` | Another run is active or stale lock exists. | `terraform plan -lock-timeout=10m`, `terraform force-unlock <id>` | Wait for active run or force-unlock only after verification. |
| `No valid credential sources found` | Provider cannot find credentials. | `terraform providers`, cloud CLI identity command | Configure OIDC, assumed role, managed identity, or environment variables. |
| `AccessDenied` / `Forbidden` | Missing IAM or wrong account. | `aws sts get-caller-identity`, `az account show`, `gcloud config get-value project` | Fix role, policy, provider alias, subscription, or project. |
| `Provider produced inconsistent result` | Provider bug or API eventual consistency. | `TF_LOG=DEBUG terraform apply`, `terraform version` | Retry if safe, pin/upgrade/downgrade provider, open provider issue. |
| `Cycle` | Circular dependency in graph. | `terraform graph`, `terraform validate` | Remove unnecessary dependencies or split stacks. |
| `Invalid index` | Wrong map/list key or count index. | `terraform console`, `terraform state list` | Validate input shape and use stable `for_each` keys. |
| `Unsupported attribute` | Output or object structure changed. | `terraform console`, `terraform output -json` | Update references or module output contracts. |
| `Resource already exists` | Resource exists outside state. | `terraform import`, `terraform state list` | Import existing object or change name. |
| `Saved plan is stale` | State changed after plan was generated. | `terraform plan -out=tfplan` | Regenerate plan and reapprove. |
| `Error loading state` | Backend unavailable or corrupt state. | `terraform state pull`, backend console/version history | Restore backend access or previous state version. |
| `Plugin did not respond` | Provider crash, network issue, or incompatible plugin. | `TF_LOG=DEBUG terraform plan`, `terraform init -upgrade` | Pin known-good provider or upgrade after testing. |
| `Invalid for_each argument` | Keys depend on unknown apply-time values. | `terraform console`, `terraform plan` | Use known static map keys or split apply into layers. |
| `Backend configuration changed` | Backend block changed since init. | `terraform init -reconfigure` | Reinitialize after confirming target backend. |
| `Error installing provider` | Registry/network/proxy issue. | `terraform init`, provider mirror logs | Use provider mirror/cache and check network proxy. |
| `Quota exceeded` | Cloud account limit reached. | Cloud quota commands and `terraform plan` | Request quota increase or reduce resource count. |

## Production-safe command recipes

### Safe production plan

```bash
terraform fmt -check -recursive
terraform validate
terraform workspace show
terraform plan -var-file=prod.tfvars -out=tfplan
terraform show tfplan
```

### Apply reviewed production plan

```bash
terraform apply tfplan
```

### Drift-only investigation

```bash
terraform plan -refresh-only -out=refresh.tfplan
terraform show refresh.tfplan
```

### State backup before risky operation

```bash
terraform state pull > state-backup-$(date +%Y%m%d-%H%M%S).json
```

### Debug provider issue into a log file

```bash
TF_LOG=DEBUG TF_LOG_PATH=terraform-debug.log terraform plan
```

### Lower parallelism for throttling

```bash
terraform plan -parallelism=5 -out=tfplan
terraform apply -parallelism=5 tfplan
```

### Move state during refactor

```bash
terraform state mv 'module.old.aws_instance.app' 'module.new.aws_instance.app'
terraform plan
```

### Safer replacement preview

```bash
terraform plan -replace='aws_instance.app' -out=replace.tfplan
terraform show replace.tfplan
```

## Interview-ready STAR answer template

```text
Situation: Terraform apply failed in production because the remote state was locked after a failed CI job.
Task: I had to restore deployment capability without corrupting state or interrupting another active run.
Action: I checked CI logs, verified no apply was running, used lock timeout first, then force-unlocked the stale lock with approval. I also added CI concurrency controls.
Result: The team resumed safe Terraform operations, avoided state corruption, and reduced repeat lock incidents.
```

## Operator checklist before production apply

- Confirm branch, workspace, backend, cloud account, region, and variable file.
- Run `terraform fmt -check -recursive` and `terraform validate`.
- Generate `terraform plan -out=tfplan` and review destructive changes.
- Store and apply the exact reviewed plan.
- Protect state, plan JSON, debug logs, and outputs because they may contain sensitive values.
- Do not use `-target`, `-lock=false`, `force-unlock`, `state rm`, or `state push` unless there is an approved runbook.
- Verify backups and rollback or forward-fix strategy for critical resources.
