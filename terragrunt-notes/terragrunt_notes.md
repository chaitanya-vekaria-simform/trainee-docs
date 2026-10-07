# Terragrunt — DevOps Notes

> Thin wrapper over Terraform / OpenTofu that keeps IaC **DRY** across many environments, regions and subscriptions/accounts.
> Examples lean on **Azure (azurerm)** with AWS notes where behaviour differs.
> Version note: Terragrunt went through a large CLI redesign in 2025 (`run --all`, `TG_*` env vars, `root.hcl`, stacks). Old syntax still shows up in most blogs — both are covered below. Confirm flags with `terragrunt --help` for your pinned version.

---

## 1. Why Terragrunt exists

Plain Terraform at scale ends up like this:

```
envs/
├── dev/
│   ├── backend.tf      ◄── copy-pasted, only "key" differs
│   ├── provider.tf     ◄── copy-pasted, only subscription differs
│   ├── main.tf         ◄── module "vnet" { source = "../../modules/vnet" ... }
│   └── variables.tf    ◄── copy-pasted
├── staging/            ◄── same 4 files again
└── prod/               ◄── same 4 files again
```

Problems:

| Pain                               | What happens in Terraform                                  | Terragrunt fix                          |
|------------------------------------|------------------------------------------------------------|-----------------------------------------|
| Backend blocks can't use variables | Hardcode state key per folder, copy-paste errors overwrite state | `remote_state` + `path_relative_to_include()` |
| Provider config repeated           | N copies of `provider.tf`                                  | `generate` block in root config         |
| One giant state per env            | Big blast radius, slow plans, lock contention              | Small *units*, one state each           |
| Cross-state wiring                 | `terraform_remote_state` data sources everywhere           | `dependency` blocks                     |
| Order of applies                   | Humans/scripts decide order                                | DAG via `run --all`                     |
| Same module, many envs             | Copy `main.tf` per env                                     | `terraform { source = ... }` + `inputs` |

Mental model in one line: **Terraform = the engine, modules = the parts, Terragrunt = the assembly line + wiring harness.**

---

## 2. Core mental model

```
                    ┌──────────────────────────────────────────┐
                    │              root.hcl                     │
                    │  remote_state  (backend, key per path)    │
                    │  generate      (provider.tf, versions.tf) │
                    │  locals/inputs (common tags, region...)   │
                    └───────────────────▲──────────────────────┘
                                        │ include "root"
            ┌───────────────────────────┼───────────────────────────┐
            │                           │                           │
   ┌────────┴────────┐        ┌─────────┴────────┐        ┌─────────┴────────┐
   │ dev/network/    │        │ dev/aks/         │        │ dev/postgres/    │
   │ terragrunt.hcl  │◄───────│ terragrunt.hcl   │        │ terragrunt.hcl   │
   │ (unit)          │ depends│ dependency "net" │        │ dependency "net" │──┐
   └────────┬────────┘        └─────────┬────────┘        └─────────┬────────┘  │
            │ source                    │ source                    │ source    │
            ▼                           ▼                           ▼           │
   modules/vnet (git tag)      modules/aks (git tag)      modules/pg (git tag)  │
            │                                                                   │
            └───────────────────────────── outputs ─────────────────────────────┘

   Each unit  =  1 module instance  =  1 state file  =  1 lock
```

Terminology (current docs):

- **Unit** — a folder with a `terragrunt.hcl`; one module instance, one state.
- **Stack** — a collection of units (a directory tree, or a `terragrunt.stack.hcl` that generates units).
- **Root config** — shared config included by every unit (`root.hcl` is the current convention; older repos use a root `terragrunt.hcl`).

### What Terragrunt actually does on `terragrunt plan`

```
 terragrunt plan  (in dev/aks/)
   │
   ├─1─► parse terragrunt.hcl  → resolve include(s), locals, functions
   ├─2─► resolve dependency blocks → run `output -json` on deps (or read state / mocks)
   ├─3─► download `source` into .terragrunt-cache/<hash>/<hash>/
   ├─4─► copy local *.tf / lock file from unit dir into cache dir
   ├─5─► write generate blocks (backend.tf, provider.tf) into cache dir
   ├─6─► export inputs as TF_VAR_<name> env vars
   ├─7─► run before_hooks
   ├─8─► (auto) terraform init  — if needed
   ├─9─► terraform plan          — working dir = cache dir
   └─10► run after_hooks / error_hooks, copy .terraform.lock.hcl back
```

Knowing step 3–6 explains most "weird" behaviour (relative paths, stale cache, lock files, inputs being ignored).

---

## 3. Recommended repo layout

```
infra-live/                          ◄── "live" repo: only terragrunt.hcl files, no resources
├── root.hcl                         ◄── backend + provider generation + global locals
├── _envcommon/                      ◄── shared per-component config (DRY across envs)
│   ├── network.hcl
│   ├── aks.hcl
│   └── postgres.hcl
├── dev/
│   ├── env.hcl                      ◄── env = "dev", subscription_id, sizes
│   └── centralindia/
│       ├── region.hcl               ◄── location = "centralindia"
│       ├── network/terragrunt.hcl
│       ├── aks/terragrunt.hcl
│       └── postgres/terragrunt.hcl
├── stage/ ...
└── prod/
    ├── env.hcl
    ├── centralindia/ ...
    └── southindia/ ...              ◄── DR region, same units, different region.hcl

infra-modules/                       ◄── separate repo, versioned with git tags
├── vnet/
├── aks/
└── postgres/
```

Rule of thumb: **modules repo = "what"**, **live repo = "where / how big / which version"**. Promote by bumping `?ref=` from dev → stage → prod.

```
  infra-modules            infra-live
  ─────────────            ─────────────────────────────────────────
  aks v1.4.0  ───────────► dev/aks     ref=v1.4.0   (test here first)
                           stage/aks   ref=v1.3.2
                           prod/aks    ref=v1.3.2
                    later: stage → v1.4.0, then prod → v1.4.0
```

---

## 4. The building blocks (with examples)

### 4.1 root.hcl — backend + provider generation (Azure)

```hcl
# root.hcl
locals {
  env_vars    = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  region_vars = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  env             = local.env_vars.locals.env
  subscription_id = local.env_vars.locals.subscription_id
  location        = local.region_vars.locals.location
}

remote_state {
  backend = "azurerm"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    resource_group_name  = "rg-tfstate-${local.env}"
    storage_account_name = "sttfstate${local.env}001"
    container_name       = "tfstate"
    key                  = "${path_relative_to_include()}/terraform.tfstate"
    use_azuread_auth     = true          # RBAC on blob, no storage keys
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "azurerm" {
  features {}
  subscription_id = "${local.subscription_id}"
}
EOF
}

generate "versions" {
  path      = "versions.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  required_version = ">= 1.6"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" }
  }
}
EOF
}

inputs = {
  location = local.location
  tags = {
    env        = local.env
    managed_by = "terragrunt"
  }
}
```

State key result:

```
unit path (relative to root.hcl)      →  blob key
dev/centralindia/network              →  dev/centralindia/network/terraform.tfstate
dev/centralindia/aks                  →  dev/centralindia/aks/terraform.tfstate
```

AWS equivalent: `backend = "s3"` with `bucket`, `key`, `region`, `encrypt = true`, and `use_lockfile = true` (S3-native locking in Terraform ≥ 1.10) or `dynamodb_table` (older). Terragrunt can **bootstrap** the S3 bucket / GCS bucket for you; for Azure, plan on pre-creating the storage account (bootstrap support for azurerm is newer/limited — check your version).

### 4.2 Unit — terragrunt.hcl

```hcl
# dev/centralindia/aks/terragrunt.hcl
include "root" {
  path   = find_in_parent_folders("root.hcl")
  expose = true                          # lets you read include.root.locals
}

include "envcommon" {
  path           = "${dirname(find_in_parent_folders("root.hcl"))}/_envcommon/aks.hcl"
  merge_strategy = "deep"
}

terraform {
  source = "git::git@github.com:org/infra-modules.git//aks?ref=v1.4.0"
}

dependency "network" {
  config_path = "../network"

  mock_outputs = {
    aks_subnet_id = "/subscriptions/0000/resourceGroups/mock/providers/Microsoft.Network/virtualNetworks/mock/subnets/mock"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan", "init"]
}

inputs = {
  cluster_name   = "aks-${include.root.locals.env}-cin"
  subnet_id      = dependency.network.outputs.aks_subnet_id
  node_count     = 2
  node_vm_size   = "Standard_D4s_v5"
}
```

### 4.3 `terraform { source }` — the double slash

```
git::git@github.com:org/infra-modules.git//aks?ref=v1.4.0
└──────────── repo ──────────────────────┘└sub┘└─ pin ─┘
```

- `//` = "download the **whole repo**, then cd into `aks`". Keeps `../common` relative module refs inside the repo working.
- Always pin `?ref=` to a **tag** (or commit SHA) in stage/prod. Never `main`.
- Local dev: `--source ../../infra-modules//aks` (old: `--terragrunt-source`) overrides source without editing files.

### 4.4 `include` + merge behaviour

```
 child terragrunt.hcl
   include "root"       (root.hcl)
   include "envcommon"  (_envcommon/aks.hcl)
          │
          ▼
 effective config = root  ⊕  envcommon  ⊕  child      (later wins)

 merge_strategy:
   "shallow" (default) → top-level blocks/attrs replaced wholesale
   "deep"              → maps (inputs, tags) merged key-by-key
   "no_merge"          → only read via expose, not merged
```

Gotcha: with shallow merge, a child `inputs = { tags = {...} }` **replaces** the root's `tags` map entirely. Use `deep` or `merge()` explicitly.

### 4.5 `dependency` vs `dependencies`

```hcl
dependency "network" {           # ordering + read outputs
  config_path = "../network"
}

dependencies {                    # ordering only, no outputs
  paths = ["../keyvault", "../dns"]
}
```

```
 run --all apply ordering (DAG)

   [network] ─────┬────► [aks] ─────► [ingress]
                  │
   [keyvault] ────┼────► [postgres]
                  │
   [dns] ─────────┘

   Level 1: network, keyvault, dns     (parallel)
   Level 2: aks, postgres              (parallel)
   Level 3: ingress
   destroy = exact reverse
```

### 4.6 Hooks

```hcl
terraform {
  source = "..."

  before_hook "tflint" {
    commands = ["plan"]
    execute  = ["tflint", "--minimum-failure-severity=error"]
  }

  after_hook "notify" {
    commands     = ["apply"]
    execute      = ["bash", "-c", "echo applied ${path_relative_to_include()}"]
    run_on_error = false
  }

  error_hook "unlock_hint" {
    commands  = ["apply", "plan"]
    execute   = ["echo", "If state is locked, check for a stuck pipeline before force-unlock"]
    on_errors = [".*state blob is already locked.*", ".*Error acquiring the state lock.*"]
  }

  extra_arguments "parallel" {
    commands  = ["apply", "plan", "destroy"]
    arguments = ["-parallelism=5"]           # Azure ARM throttling (429s)
  }
}
```

Hooks run in the **cache dir**, not your unit dir — use `get_terragrunt_dir()` for paths back to source.

### 4.7 Retries / errors

```hcl
errors {
  retry "transient_azure" {
    retryable_errors = [
      "(?s).*TooManyRequests.*",
      "(?s).*Status=429.*",
      "(?s).*context deadline exceeded.*",
    ]
    max_attempts       = 3
    sleep_interval_sec = 30
  }
  ignore "known_safe" {
    ignorable_errors = ["(?s).*some noisy but harmless error.*"]
  }
}
```

(Older configs: `retryable_errors`, `retry_max_attempts`, `retry_sleep_interval_sec` at top level.)

### 4.8 Useful built-in functions

| Function                                | Returns                                              |
|-----------------------------------------|------------------------------------------------------|
| `find_in_parent_folders("x.hcl")`       | Abs path to nearest parent `x.hcl` (errors if none)  |
| `path_relative_to_include()`            | Unit path relative to the included (root) config     |
| `get_terragrunt_dir()`                  | Dir of the *current* terragrunt.hcl                  |
| `get_parent_terragrunt_dir()`           | Dir of the included (parent) config                  |
| `get_repo_root()`                       | Git repo root                                        |
| `read_terragrunt_config(path)`          | Parse another HCL file (locals/inputs) as an object  |
| `get_env("NAME", "default")`            | Env var                                              |
| `run_cmd("--terragrunt-quiet", "az", ...)` | Shell out (runs on **every parse** — careful)     |
| `sops_decrypt_file("secrets.enc.yaml")` | Decrypt SOPS file (Azure Key Vault / KMS keys)       |
| `get_terraform_commands_that_need_vars()` | List for `extra_arguments` with `-var-file`        |

---

## 5. Stacks (newer feature)

Instead of hand-maintaining dozens of unit folders, a `terragrunt.stack.hcl` **generates** them.

```hcl
# prod/centralindia/terragrunt.stack.hcl
unit "network" {
  source = "git::git@github.com:org/infra-catalog.git//units/network?ref=v2.0.0"
  path   = "network"
  values = { cidr = "10.20.0.0/16" }
}

unit "aks" {
  source = "git::git@github.com:org/infra-catalog.git//units/aks?ref=v2.0.0"
  path   = "aks"
  values = { node_count = 5, network_path = "../network" }
}
```

```
 terragrunt stack generate
        │
        ▼
 .terragrunt-stack/
   ├── network/terragrunt.hcl   (reads values.cidr)
   └── aks/terragrunt.hcl       (reads values.node_count)
```

- Units read inputs via `values.*`.
- Generated dir is build output — usually gitignored.
- Good for "stamp out same platform per region/customer". For a small repo, plain folders are simpler.

---

## 6. CLI cheat sheet (new vs old)

| Task                               | Current CLI                                   | Old CLI (still in blogs)                 |
|------------------------------------|-----------------------------------------------|------------------------------------------|
| Plan one unit                      | `terragrunt plan`                             | same                                     |
| Pass TF flags explicitly           | `terragrunt run -- plan -lock-timeout=5m`     | `terragrunt plan -lock-timeout=5m`       |
| Plan everything under cwd          | `terragrunt run --all plan`                   | `terragrunt run-all plan`                |
| Apply everything (CI)              | `terragrunt run --all --non-interactive apply`| `terragrunt run-all apply --terragrunt-non-interactive` |
| Apply unit + its deps              | `terragrunt run --graph apply`                | `terragrunt apply --terragrunt-include-external-dependencies` |
| Show DAG                           | `terragrunt dag graph \| dot -Tpng > g.png`   | `terragrunt graph-dependencies`          |
| List units                         | `terragrunt find` / `terragrunt list`         | —                                        |
| Format HCL                         | `terragrunt hcl fmt`                          | `terragrunt hclfmt`                      |
| Validate inputs vs module vars     | `terragrunt hcl validate --inputs`            | `terragrunt validate-inputs`             |
| See merged config                  | `terragrunt render --format json`             | `terragrunt render-json`                 |
| Local module override              | `--source ../modules//aks`                    | `--terragrunt-source`                    |
| Limit concurrency                  | `--parallelism 4`                             | `--terragrunt-parallelism 4`             |
| Use OpenTofu/Terraform binary      | `--tf-path tofu` / `TG_TF_PATH`               | `--terragrunt-tfpath` / `TERRAGRUNT_TFPATH` |
| Debug logs                         | `--log-level debug`                           | `--terragrunt-log-level debug`           |
| Provider cache (big speedup)       | `--provider-cache`                            | `TERRAGRUNT_PROVIDER_CACHE=1`            |

Env vars moved from `TERRAGRUNT_*` → `TG_*` (e.g. `TG_NON_INTERACTIVE=true`).

---

## 7. CI/CD flow

```
 PR opened
   │
   ├─► fmt check:     terragrunt hcl fmt --check
   ├─► lint:          tflint / checkov / trivy on modules
   ├─► detect changed units (git diff → dirs containing terragrunt.hcl)
   ├─► plan:          terragrunt run --all --non-interactive \
   │                    --queue-include-dir <changed>  plan -out=tfplan
   ├─► post plan summary as PR comment
   │
 PR approved + merged
   │
   ├─► (prod) manual approval gate (environment protection)
   └─► apply saved plans in DAG order
        terragrunt run --all --non-interactive apply tfplan
```

Auth in Azure DevOps / GitHub Actions: Workload Identity Federation (OIDC) → `ARM_USE_OIDC=true`, `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`. No client secrets in pipelines.

Pipeline must-haves:

- Pin both versions: Terragrunt **and** Terraform/OpenTofu (`.terragrunt-version`, `.terraform-version` / mise / asdf / container image).
- `--non-interactive` — otherwise `run --all` waits for "Are you sure? (y/n)" forever.
- Provider cache dir on a persisted volume/cache step.
- Separate service principals/identities per env; prod identity only on the prod pipeline.
- Serialize applies per env (concurrency group) — two pipelines on same unit = lock fights.

---

## 8. Gotchas (the stuff that bites in real life)

### Inputs & variables

1. **Typo in `inputs` = silently ignored.** Inputs become `TF_VAR_x`; if the module has no `variable "x"`, Terraform doesn't care. Use `terragrunt hcl validate --inputs` in CI.
2. Complex types (maps/lists/objects) are JSON-encoded into env vars — fine, but **`number` vs `string`** mismatches surface as odd errors. Keep module variable types explicit.
3. `TF_VAR_*` already set in your shell (from another project) **overrides nothing** — Terragrunt sets them — but `*.auto.tfvars` in the unit dir gets copied into cache and can conflict. Keep unit dirs clean.

### Dependencies & mocks

4. **Mocks leaking into apply.** If `mock_outputs_allowed_terraform_commands` includes `apply` (or is unset in older versions with `skip_outputs`), you can apply a fake subnet ID. Only allow `validate`/`plan`/`init`.
5. `run --all plan` on a brand-new env: downstream plans use mocks → plan output is **not** what apply will do. Apply level by level for the first rollout.
6. Every `dependency` triggers `terraform output` on that unit (init + state read). 30 units × 5 deps = slow. Remote-state optimization (read outputs directly from backend) helps; keep the dependency graph shallow.
7. Renaming/moving a unit folder **changes its state key** (`path_relative_to_include`). Terraform sees an empty state → wants to recreate everything. Move the state blob first (or `terraform state` pull/push) before merging the rename.

### Cache & paths

8. `.terragrunt-cache/` grows huge (provider binaries per unit). Gitignore it; clean with `find . -type d -name .terragrunt-cache -prune -exec rm -rf {} +`. Use the provider cache.
9. Stale cache after changing `source` ref or switching branches → weird diffs. Delete cache, or `--source-update`.
10. Relative paths in modules (`file("${path.module}/x.json")` is fine; `file("../x.json")` breaks) because code runs from cache dir.
11. `.terraform.lock.hcl` is generated in cache and **copied back** to the unit dir. Commit it per unit, or providers drift between CI and laptops. Multi-platform: `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64`.
12. `get_terragrunt_dir()` inside root.hcl returns the **child's** dir (evaluated in child context). Use `get_parent_terragrunt_dir()` for root-relative paths.

### Includes & locals

13. Child can't see root `locals` unless `expose = true` → `include.root.locals.x`.
14. Only **one level** of `include` historically — an included file can't include another. Use `read_terragrunt_config()` for chaining.
15. `run_cmd()` in locals runs on **every parse**, including during `run --all` for every unit and dependency evaluation. An `az` call there = dozens of calls. Cache it (`--terragrunt-global-cache` arg) or avoid.

### Run-all

16. `run --all destroy` from the wrong directory = destroys more than you think. It's scoped to cwd **plus** it follows dependencies. Always `terragrunt find`/`dag graph` first; use `--queue-exclude-dir` / `exclude` blocks; put `prevent_destroy = true` in prod stateful units.
17. One unit failing → dependents are skipped, siblings keep going. Read the summary at the end, not just the last error.
18. High parallelism + Azure → ARM `429 TooManyRequests`. Lower `--parallelism` and add retry rules.
19. Interactive prompt for each external dependency outside cwd — CI hangs without `--non-interactive`.

### State & backend

20. Azure: Terragrunt won't create the storage account for you (in most versions) — chicken-and-egg. Bootstrap state storage with a small separate Terraform/Bicep/`az` script, with versioning + soft delete + resource lock.
21. `if_exists = "overwrite_terragrunt"` only overwrites files *Terragrunt generated*. If someone committed a hand-written `backend.tf` in a unit dir, you get an error — that's protecting you; delete the manual file.
22. Changing backend config (e.g. new storage account) → needs `init -migrate-state`. Terragrunt's auto-init won't migrate silently.

### Versions

23. Copy-pasted blog commands (`run-all`, `--terragrunt-*`) produce deprecation warnings or fail on new versions — and new syntax fails on old versions. Pin and standardize.

---

## 9. Terragrunt vs alternatives (quick take)

| Option                          | When it fits                                                     |
|---------------------------------|------------------------------------------------------------------|
| Plain Terraform + workspaces    | Few envs, identical config, one team. Workspaces share backend config/code — risky for prod isolation. |
| Plain Terraform + folders + tfvars | Small/medium; accept some duplication                         |
| **Terragrunt**                  | Many envs × regions × subscriptions, many small states, need DAG |
| Terraform Stacks (HCP)          | HCP Terraform users wanting native multi-deployment              |
| Terramate / Atmos               | Similar orchestration; Terramate has strong change detection, Atmos is YAML-config-driven |
| Spacelift / env0 / Atlantis     | Orchestration/PR automation *around* Terraform or Terragrunt (complementary) |

Don't adopt Terragrunt for a 1-env, 10-resource project — it's overhead.

---

## 10. Production scenario FAQ

**Q1. You ran `terragrunt run --all plan` for a new `stage` env and plan looks clean, but `apply` failed on AKS with "subnet not found". Why?**
Plan used `mock_outputs` for the network dependency because network had no state yet. The real subnet didn't exist until network applied. Fix: apply in levels for first rollout (`network` → then rest), or use `run --all apply` (Terragrunt applies network first and reads real outputs). Never trust a `run --all plan` on an empty env.

**Q2. A teammate renamed `dev/cin/postgres` to `dev/centralindia/postgres`. The PR plan shows "destroy 0, create 14" for postgres. What happened and what do you do?**
State key derived from path changed → new empty state. Do NOT apply. Copy the old blob to the new key (`az storage blob copy start ...` or `terraform state pull` from old / `push` to new), re-plan, expect "no changes", then delete the old blob. Add a CI check that blocks apply when a unit plan shows only creates for an existing env.

**Q3. Prod apply is stuck: "state blob is already locked".**
Check the lock info (who/when/operation). Confirm no pipeline/laptop is still running (Azure DevOps runs, GitHub Actions, colleagues). Only then `terragrunt run -- force-unlock <LOCK_ID>` in that unit (azurerm: lock = blob lease; can also break lease in portal). Root cause usually a cancelled pipeline — add concurrency groups so only one apply per env runs.

**Q4. CI takes 25 minutes for `run --all plan` across 60 units. How to speed it up?**
(a) Plan only changed units + their dependents (git diff → `--queue-include-dir`). (b) Enable provider cache server + cache the dir in CI. (c) Ensure dependency output reads from remote state rather than full init. (d) Keep dependency graph shallow. (e) Raise `--parallelism` within ARM throttling limits. (f) Split pipelines per env.

**Q5. The `tags` you added in a unit wiped the `managed_by` and `env` tags from root. Why?**
Default shallow merge: child `inputs.tags` replaced root's `inputs.tags`. Use `merge_strategy = "deep"` on the include, or `tags = merge(include.root.inputs.tags, { team = "data" })` with `expose = true`.

**Q6. Module v1.5.0 has a breaking change. How do you roll it out safely?**
Bump `?ref=v1.5.0` in dev only → plan/apply → soak → stage → prod, each its own PR. If `_envcommon` sets the source, parametrize the version per env (`local.env_vars.locals.aks_module_version`) so one PR doesn't bump all envs at once. Use `moved {}` blocks in the module for renames to avoid recreation.

**Q7. Someone ran `terragrunt run --all destroy` in `dev/` thinking it'd only kill one service. How do you prevent this?**
Guardrails: `prevent_destroy = true` on stateful units (DBs, Key Vault, storage), Azure resource locks (`CanNotDelete`) on prod RGs, least-privilege identities (laptops get Reader on stage/prod), destroy only via pipeline with approval, and train people to `terragrunt find` / `dag graph` before any `--all`. Restore path: Key Vault/Storage soft delete, PostgreSQL PITR, state blob versioning.

**Q8. Plans differ between a dev's laptop and CI for the same commit.**
Usual suspects: different Terraform/Terragrunt versions, uncommitted/missing `.terraform.lock.hcl`, stale `.terragrunt-cache` with old module ref, local `--source` override left in an env var (`TG_SOURCE`), different `ARM_SUBSCRIPTION_ID` in shell. Pin versions, commit lock files, clear cache, `env | grep -E 'TG_|TERRAGRUNT|ARM_'`.

**Q9. How do you handle secrets (DB admin password, API keys)?**
Best: don't pass them at all — module generates (`random_password`) and writes to Key Vault; consumers read from Key Vault. If you must pass: `sops_decrypt_file()` with Azure Key Vault-backed SOPS keys, or pipeline secret → `TF_VAR_` env. Remember **state still contains secrets** — lock down state storage (private endpoint, RBAC, no shared keys).

**Q10. Two regions (Central India primary, South India DR). How do you structure it?**
Same units under `prod/centralindia/` and `prod/southindia/`, different `region.hcl` (location, CIDRs). State keys separate by path. Cross-region dependencies (e.g. global Front Door needs both AKS ingress IPs) live in a `prod/_global/` unit that depends on both. Ensure the state storage itself is GRS/RA-GRS or replicated, else DR of infra code fails with the primary region.

**Q11. You need to import an existing manually-created VNet into a Terragrunt-managed unit.**
Preferred: add an `import {}` block in the module or a generated `imports.tf` (`generate` block) → `terragrunt plan` shows import → apply → remove block. CLI way: `terragrunt run -- import 'azurerm_virtual_network.this' <resource_id>` from the unit dir (uses the same inputs/backend).

**Q12. `terragrunt apply` in a unit asks for confirmation to apply external dependency `../network` too. Is that bad?**
That's `--graph`/include-external-dependencies behaviour or a run-all reaching outside cwd. In CI answer explicitly via flags; locally, say **no** unless you intend to touch network. Prefer per-unit applies for targeted changes.

**Q13. Provider upgrade (azurerm 3.x → 4.x) across 60 units — how?**
Change the version constraint in the root `generate "versions"` block, but roll it out per env: parametrize via `env.hcl`. Run `terragrunt run --all -- init -upgrade` in dev, fix breaking changes (v4 requires `subscription_id`, resource provider registration changes), commit regenerated lock files, then promote.

**Q14. Drift detection — how do you know someone clicked in the portal?**
Scheduled pipeline: `terragrunt run --all --non-interactive plan -detailed-exitcode` (exit 2 = changes) → alert to Teams/Slack with the unit list. Pair with Azure Policy + Activity Log alerts on write ops by non-pipeline identities.

---

## 11. Quick checklist for a new Terragrunt repo

```
[ ] Pin terragrunt + terraform/tofu versions (CI + local)
[ ] root.hcl: remote_state with path_relative_to_include(), generate provider/versions
[ ] env.hcl / region.hcl hierarchy, no hardcoded subscription IDs in units
[ ] Modules in separate repo, tagged, pinned with ?ref=
[ ] mock_outputs only for validate/plan/init
[ ] .terragrunt-cache/ in .gitignore; .terraform.lock.hcl committed
[ ] hcl fmt --check + hcl validate --inputs in CI
[ ] Plan changed units only; apply with approval gates for prod
[ ] OIDC auth, per-env identities
[ ] prevent_destroy + Azure resource locks on stateful prod resources
[ ] State storage: versioning, soft delete, RBAC auth, private endpoint
[ ] Concurrency control: one apply per env at a time
[ ] Scheduled drift detection
```
