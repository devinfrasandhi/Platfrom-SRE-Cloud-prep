# Terraform — Interview Prep

## Refresher

### What it is
Terraform is a declarative Infrastructure-as-Code (IaC) tool by HashiCorp. You describe the desired end state in HCL (HashiCorp Configuration Language), and Terraform computes the difference between that and reality, then makes API calls to reconcile them. It is cloud-agnostic via **providers** (AWS, Azure, GCP, Kubernetes, Datadog, etc.).

### Core workflow
```
terraform init      # download providers/modules, configure backend
terraform validate  # syntax + internal consistency check
terraform plan      # compute the diff (create/update/destroy)
terraform apply     # execute the plan
terraform destroy   # tear everything down
```

### Key concepts you must be fluent in

**State** — Terraform's record of what it manages, mapping config resources to real-world objects. Stored in `terraform.tfstate`. In teams, state lives in a **remote backend** (S3, Azure Blob, GCS, Terraform Cloud) with **state locking** (S3 now supports native locking via `use_lockfile`; DynamoDB was the classic approach) to prevent concurrent corruption. State can contain secrets in plaintext — treat it as sensitive, encrypt at rest, restrict access.

**Providers** — plugins that translate HCL into API calls. Pin versions with `required_providers` to avoid surprise breaking changes.

**Resources vs Data Sources** — `resource` blocks create/manage things. `data` blocks read existing things you don't manage (e.g., look up the latest AMI, an existing VPC).

**Variables, Outputs, Locals**
- `variable` — inputs (with types, defaults, validation blocks, `sensitive = true`)
- `output` — exported values, consumable by other configs or humans
- `locals` — computed intermediate values to DRY up config

**Modules** — reusable packages of Terraform config. A root module calls child modules. Best practice: version-pinned modules from a registry or Git tags, small and composable, clear input/output contracts.

**Meta-arguments**
- `count` — create N copies, indexed numerically (fragile: removing item 0 shifts everything)
- `for_each` — create copies keyed by map/set keys (stable addressing, preferred)
- `depends_on` — explicit dependency when Terraform can't infer it
- `lifecycle` — `create_before_destroy`, `prevent_destroy`, `ignore_changes`
- `provider` — pin a resource to an aliased provider (e.g., multi-region)

**Workspaces** — multiple state files for the same config. Fine for lightweight variation; most teams prefer separate directories or separate state per environment for prod isolation.

**Provisioners** — `local-exec` / `remote-exec` run scripts. Official stance: last resort. Prefer cloud-init/user_data, config management, or building images with Packer.

### State surgery commands (interviewers love these)
```
terraform state list                  # what's in state
terraform state show <addr>           # details of one resource
terraform state mv <src> <dst>        # rename/move without destroy
terraform state rm <addr>             # forget a resource (doesn't delete it)
terraform import <addr> <id>          # adopt existing infra into state
terraform apply -replace=<addr>       # force recreation (replaces deprecated taint)
terraform plan -refresh-only          # detect drift without proposing changes
terraform force-unlock <lock-id>      # break a stuck state lock
```
Modern alternative to CLI surgery: `import`, `moved`, and `removed` **blocks** in code — declarative, reviewable in PRs.

---

## Interview Questions

### Warm-up tier

**Q: What is state and why does Terraform need it?**
A: State maps configuration to real-world resources, stores resource metadata and dependencies, and caches attributes for performance. Without it, Terraform would have to interrogate the entire cloud account on every run and couldn't reliably know which resources it owns versus ones created elsewhere.

**Q: `count` vs `for_each` — when and why?**
A: Both create multiple instances. `count` indexes numerically (`aws_instance.web[0]`), so removing an element from the middle of the list shifts indices and Terraform wants to destroy/recreate everything after it. `for_each` keys by map key or set value (`aws_instance.web["frontend"]`), so adding/removing one item touches only that item. Default to `for_each` for anything with identity; `count` is fine for "N identical copies" or as a boolean toggle (`count = var.enabled ? 1 : 0`).

**Q: Difference between `terraform plan` and `apply`? Can apply do something plan didn't show?**
A: Plan computes the diff; apply executes it. Yes, they can diverge — if infrastructure changes between plan and apply, or if you run plain `apply` (which re-plans). That's why CI pipelines save the plan file (`terraform plan -out=tfplan`) and apply exactly that artifact (`terraform apply tfplan`).

### Complex tier — these separate candidates

**Q1: Two engineers ran `apply` simultaneously and now state is corrupted / doesn't match reality. Walk me through recovery.**
A: First, stop all further applies and confirm locking is enabled going forward (this happened because it wasn't, or was force-unlocked carelessly). Recovery: (1) Pull the current state (`terraform state pull > backup.tfstate`) and back it up. (2) If using S3 versioning, inspect previous state versions to find the last known-good one. (3) Run `terraform plan -refresh-only` to see what Terraform thinks drifted versus reality. (4) Reconcile: use `terraform import` for real resources missing from state, `terraform state rm` for state entries whose resources are gone, and `terraform apply -refresh-only` to accept attribute drift. (5) Iterate until `terraform plan` shows a clean or expected diff. Root-cause fix: remote backend with locking mandatory, applies only through CI with serialized runs.

**Q2: You need to rename a resource (or refactor it into a module) without destroying it. How?**
A: Terraform tracks resources by address, so renaming looks like "destroy old, create new." Two options: `terraform state mv 'aws_db_instance.old' 'module.db.aws_db_instance.main'` (imperative, immediate), or better, a `moved` block in code:
```hcl
moved {
  from = aws_db_instance.old
  to   = module.db.aws_db_instance.main
}
```
The `moved` block is reviewable in a PR, applies for everyone using the config, and is the modern best practice. Verify with `terraform plan` showing no changes (or only the move).

**Q3: Your state file is 8MB, plans take 15 minutes, and one team's mistake can hit another team's infra. Restructure it.**
A: Classic monolithic state problem. Split by blast radius and rate of change: separate states per environment (prod/staging), and within an environment, per layer — networking (VPC, subnets), shared services (DNS, IAM), and per-team/per-service application stacks. Wire layers together with `terraform_remote_state` data sources or (looser coupling, often better) data-source lookups by tags/names. Benefits: faster plans, smaller blast radius, per-team access control on state buckets, independent apply cadence. Migration path: `terraform state mv` with `-state-out`, or state pull/edit/push per chunk, done incrementally with plans verifying zero changes after each move.

**Q4: How do you handle secrets in Terraform, given state stores values in plaintext?**
A: Layered answer. (1) Never hardcode secrets in `.tf` files. (2) Mark variables `sensitive = true` — hides from CLI output but *not* from state. (3) Since state will contain whatever secrets pass through it: encrypt state at rest (SSE-KMS on S3), tightly IAM-restrict the state bucket, enable versioning + access logging. (4) Best pattern: keep secrets out of Terraform's hands entirely — have Terraform create the *container* (e.g., an empty Secrets Manager secret) while a separate process writes the value; or generate secrets provider-side (`aws_secretsmanager_random_password`, RDS `manage_master_user_password`) so they never transit your machine. (5) Vault provider is an option, but read secrets still land in state — ephemeral values/resources (newer Terraform versions) address exactly this by keeping values out of state and plan.

**Q5: `terraform plan` shows changes nobody made. What's your diagnosis process?**
A: This is drift or a perpetual-diff bug. Diagnose: (1) Read the diff carefully — is it a real infrastructure change (someone clicked in the console) or a no-op diff (field ordering, case, defaults)? (2) `terraform plan -refresh-only` isolates pure drift from config changes. (3) Check CloudTrail/audit logs for who changed it. Handle real drift by either applying (revert the manual change) or updating config to match (accept it). Handle perpetual diffs — where every apply "fixes" it and it comes back — by checking for known provider bugs, API-side normalization (lowercased tags, reordered JSON policies — use `jsonencode` on a data source policy document), or another controller fighting Terraform (e.g., an autoscaler changing `desired_count` — fix with `lifecycle { ignore_changes = [desired_count] }`).

**Q6: Design a Terraform CI/CD pipeline for a team of 20 with prod safety.**
A: PR opens → pipeline runs `fmt -check`, `validate`, `tflint`, security scan (trivy/checkov), then `terraform plan -out` per affected stack, posting the plan as a PR comment. Merge to main → apply runs, but only the saved plan artifact, serialized per state (concurrency groups), with a manual approval gate for prod. Key properties: nobody runs apply from a laptop (CI has the only credentials, via OIDC federation — no long-lived keys), state locking as backstop, plans reviewable in PRs, drift detection via scheduled `plan -detailed-exitcode` alerting on unexpected diffs. Mention Atlantis or Terraform Cloud as prebuilt versions of this pattern.

**Q7: A module you consume released a breaking change. How does your setup ensure this doesn't hit you unaware, and how do you upgrade safely?**
A: Prevention: always pin module versions (`version = "~> 4.2"` allows patches, blocks 5.x) and provider versions, with a lockfile (`.terraform.lock.hcl`) committed. Upgrade process: read the changelog, bump in a branch, run `terraform init -upgrade` and `plan` against a non-prod state, scrutinize the diff for destroys/replaces (especially resources with `ForceNew` attribute changes), use `moved` blocks if the module refactored internal addresses, then roll env by env. If the plan wants to replace stateful resources (databases!), stop and evaluate `state mv` or import strategies rather than accepting recreation.

**Q8: Explain `create_before_destroy` and a scenario where it saves you — and one where it bites you.**
A: By default Terraform destroys then creates on replacement. `create_before_destroy = true` reverses that: new resource comes up before the old dies — essential for things like launch templates behind ASGs, TLS certs, or anything where a gap means downtime. Where it bites: resources with unique constraints — e.g., an IAM role or S3 bucket with a fixed name can't have two copies exist simultaneously, so the create fails. Fix is name-with-random-suffix patterns (`name_prefix`) or accepting destroy-first for those.

---
