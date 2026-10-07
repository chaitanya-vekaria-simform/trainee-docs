# Bicep — DevOps Notes

> Azure-native Infrastructure as Code. A friendlier language that **compiles to ARM JSON** and is deployed by Azure Resource Manager (ARM).
> Hands-on: [`../labs/bicep_practice_lab.md`](../labs/bicep_practice_lab.md) (10 exercises, all code compile-checked).

---

## 1. Mental model

```
   you write                 compile (local)               Azure does the work
 ┌────────────┐  bicep build  ┌──────────────┐  PUT deployment ┌──────────────────────┐
 │ main.bicep │ ────────────► │ main.json    │ ───────────────►│ Azure Resource       │
 │ modules/   │               │ (ARM template)│                │ Manager (ARM)        │
 │ *.bicepparam│              └──────────────┘                 │  • validates          │
 └────────────┘                                                │  • builds dependency  │
                                                               │    graph              │
   NO state file.                                              │  • calls resource     │
   Azure itself is the "state".                                │    providers in       │
                                                               │    parallel           │
                                                               │  • stores deployment  │
                                                               │    history            │
                                                               └──────────────────────┘
```

Key ideas:

- **Declarative + idempotent** — describe the end state; deploying the same template twice = no change.
- **No state file** (unlike Terraform). ARM compares the template with what exists *right now*.
- **Day-0 support** — any new Azure resource/API version is usable immediately (types come from Azure's REST specs).
- **Azure-only.** For multi-cloud, Terraform/OpenTofu.

---

## 2. Bicep vs ARM JSON vs Terraform

| Topic               | ARM JSON           | Bicep                          | Terraform (azurerm)                    |
|---------------------|--------------------|--------------------------------|----------------------------------------|
| Syntax              | Verbose JSON       | Concise DSL                    | HCL                                    |
| State               | None (Azure)       | None (Azure)                   | State file (must store + lock)         |
| New Azure features  | Day 0              | Day 0                          | Wait for provider release (or azapi)   |
| Plan/preview        | what-if            | what-if                        | `terraform plan` (more reliable)       |
| Deletes removed resources | Complete mode only | Deployment stacks / complete mode | Yes, by default                  |
| Multi-cloud / SaaS  | No                 | No (extensions: Graph, K8s - limited) | Yes                             |
| Drift detection     | Weak               | Weak (what-if)                 | Strong (`plan`)                        |
| Learning curve      | High               | Low                            | Medium                                 |

When teams pick Bicep: Azure-only shop, Microsoft partner/client requirement, want no state management, need brand-new Azure features immediately.

---

## 3. Core terms (glossary)

| Term | Meaning |
|------|---------|
| **ARM** (Azure Resource Manager) | The Azure control plane. Every portal click, CLI call, Terraform apply and Bicep deployment ends up as an ARM REST call. |
| **ARM template** | JSON file describing resources. Bicep compiles into this. |
| **Resource provider** | Service namespace, e.g. `Microsoft.Storage`, `Microsoft.Network`. Must be *registered* in the subscription. |
| **Resource type** | `Microsoft.Storage/storageAccounts` |
| **API version** | `@2023-05-01` — the schema version of the resource type. Pin it; newer ≠ always better. |
| **Symbolic name** | The Bicep-only name (`stg` in `resource stg '...'`). Used to reference the resource in code. Not the Azure name. |
| **param** | Input value. Can have type, default, decorators. |
| **var** | Computed value inside the template. |
| **output** | Value returned after deployment (to CLI, pipeline, or parent module). |
| **module** | Another `.bicep` file called from this one. Becomes a **nested deployment**. |
| **Decorator** | `@description`, `@allowed`, `@minLength`, `@secure`, `@batchSize`, `@export`... metadata/validation on params, resources, types. |
| **Scope** | Where a deployment targets: `resourceGroup` (default), `subscription`, `managementGroup`, `tenant`. Set via `targetScope`. |
| **existing** | Reference a resource already in Azure without managing it. |
| **parent** | Declare a child resource (e.g. blob service under storage account). |
| **Implicit dependency** | Created automatically when one resource references another's property (`plan.id`). Preferred. |
| **dependsOn** | Explicit dependency — only when there is no property reference. |
| **.bicepparam** | Bicep-native parameter file with a `using` line pointing to the template. Replaces `parameters.json`. |
| **what-if** | Preview of changes (`+ create`, `~ modify`, `- delete`, `= nochange`, `* ignore`). |
| **Deployment mode** | **Incremental** (default — add/update, never delete) vs **Complete** (delete anything in RG not in template — dangerous). |
| **Deployment stack** | A deployment that tracks the resources it manages; can delete removed resources and apply **deny settings** (lock). The modern replacement for complete mode / Blueprints. |
| **Deployment history** | Each deployment is saved in the scope (max 800 per RG — older ones auto-deleted). |
| **Linter** | Built-in rules (configured in `bicepconfig.json`) — e.g. no hardcoded locations, no secrets in outputs. |
| **Bicep registry** | Modules published to Azure Container Registry (`br:myacr.azurecr.io/bicep/...`) or the public registry (`br/public:`). |
| **AVM** (Azure Verified Modules) | Microsoft-maintained, tested modules in the public registry: `br/public:avm/res/...` (resource) and `avm/ptn/...` (pattern). |
| **Template spec** | A template stored as an Azure resource (`Microsoft.Resources/templateSpecs`) with versions and RBAC; shareable without a git repo. |
| **User-defined type** | `type subnetSpec = {...}` — strong typing for params; caught at compile time. |
| **User-defined function** | `func name(...) type => expr` — reusable expressions. |
| **@export / import** | Share types, functions and variables across files. |
| **Extension** (was "provider") | Lets Bicep manage non-ARM things (e.g. Microsoft Graph, Kubernetes) — still maturing. |
| **Decompile** | `az bicep decompile --file x.json` → best-effort convert ARM JSON → Bicep. Also "export template" from portal → decompile. |

---

## 4. Syntax cheat sheet

```bicep
targetScope = 'resourceGroup'               // default; or 'subscription' | 'managementGroup' | 'tenant'

// ---------- params ----------
@description('Environment name')
@allowed([ 'dev', 'prod' ])
param env string

param location string = resourceGroup().location   // never hardcode 'eastus'
@secure()
param adminPassword string                         // not logged, not in outputs
param tags object = {}
param subnets array = []
@minValue(1)
@maxValue(10)
param nodeCount int = 2

// ---------- vars ----------
var suffix   = uniqueString(resourceGroup().id)    // 13 chars, deterministic
var isProd   = env == 'prod'
var stgName  = take('st${env}${suffix}', 24)       // storage name max 24, lowercase

// ---------- resource ----------
resource stg 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: stgName
  location: location
  sku: { name: isProd ? 'Standard_ZRS' : 'Standard_LRS' }
  kind: 'StorageV2'
  tags: tags
}

// ---------- child resource ----------
resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  name: '${stg.name}/default/uploads'               // or use parent: + short name
}

// ---------- existing ----------
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'kv-shared'
  scope: resourceGroup('rg-shared')                // can be another RG / subscription
}

// ---------- condition ----------
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = if (isProd) { ... }

// ---------- loops ----------
resource nsgs 'Microsoft.Network/networkSecurityGroups@2024-05-01' = [for s in subnets: {
  name: 'nsg-${s.name}'
  location: location
}]
var names = [for (s, i) in subnets: '${i}-${s.name}']
@batchSize(2)                                       // serial-ish loop deployment
module m 'x.bicep' = [for i in range(0, 5): { name: 'm-${i}', params: {} }]

// ---------- module ----------
module net 'modules/network.bicep' = {
  name: 'network-deploy'                            // nested deployment name (unique per scope!)
  scope: resourceGroup('rg-network')
  params: { location: location }
}

// ---------- outputs ----------
output stgId string = stg.id
output subnetId string = net.outputs.subnetId

// ---------- null-safety ----------
var sku = config.?sku ?? 'Standard_LRS'             // .? safe access, ?? default
var item = myMap[?key]                              // safe index
var forced = maybeNull!                             // non-null assertion
```

Common functions:

| Function | Use |
|---|---|
| `resourceGroup()`, `subscription()`, `tenant()` | Current scope info (`.id`, `.location`, `.tenantId`) |
| `uniqueString(a, b...)` | Deterministic 13-char hash → globally unique names |
| `guid(a, b...)` | Deterministic GUID → role assignment names |
| `resourceId()`, `subscriptionResourceId()` | Build IDs for things not in the template (e.g. role definitions) |
| `union()`, `intersection()`, `contains()`, `empty()`, `length()` | Collections |
| `concat()`, `format()`, `toLower()`, `take()`, `replace()`, `split()` | Strings |
| `loadTextContent('x.sh')`, `loadJsonContent('x.json')`, `loadFileAsBase64()` | Embed files at compile time (cloud-init, policies) |
| `kv.getSecret('name')` | Pass Key Vault secret to a `@secure()` module param |
| `environment()` | Cloud endpoints (public vs China/Gov) — avoid hardcoding `core.windows.net` |
| `listKeys(stg.id, stg.apiVersion)` / `stg.listKeys()` | Fetch keys at deploy time (prefer managed identity instead) |

---

## 5. Deployment scopes

```
 Tenant  (az deployment tenant)        → management groups, subscriptions (aliases)
   └── Management group (az deployment mg)   → policies, role assignments at MG
         └── Subscription (az deployment sub -l <loc>)  → resource groups, policies, budgets
               └── Resource group (az deployment group -g <rg>)  → almost everything
```

```bash
az deployment group create -g rg-app -f main.bicep -p dev.bicepparam
az deployment sub   create -l centralindia -f main.bicep          # -l = where metadata is stored
az deployment mg    create -m mg-platform -l centralindia -f main.bicep
az deployment tenant create -l centralindia -f main.bicep
```

A subscription-scope template can create RGs and deploy modules into them with `scope: rg` — one entry point for a whole environment.

---

## 6. Recommended repo layout

```
infra/
├── bicepconfig.json                ◄── linter rules, registry aliases
├── main.bicep                      ◄── entry point (often targetScope = 'subscription')
├── modules/
│   ├── network.bicep
│   ├── aks.bicep
│   └── private-endpoint.bicep
├── types/
│   └── shared.bicep                ◄── @export() types/functions
├── params/
│   ├── dev.bicepparam
│   ├── qa.bicepparam
│   └── prod.bicepparam
└── pipelines/
    └── deploy.yml
```

`bicepconfig.json` example:

```json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-hardcoded-location": { "level": "error" },
        "outputs-should-not-contain-secrets": { "level": "error" },
        "use-recent-api-versions": { "level": "warning" },
        "no-unused-params": { "level": "warning" }
      }
    }
  },
  "moduleAliases": {
    "br": {
      "corp": { "registry": "acrplatform.azurecr.io", "modulePath": "bicep/modules" }
    }
  }
}
```

Use: `module x 'br/corp:network:1.2.0' = {...}`

---

## 7. CI/CD flow

```
 PR
  ├─► az bicep build --file main.bicep           (compile + lint, fails on errors)
  ├─► az deployment group validate ...           (ARM preflight: quota, names, policy)
  ├─► az deployment group what-if ... > whatif.txt  → post as PR comment
  │
 merge to main
  ├─► deploy dev  (az deployment group create / az stack group create)
  ├─► smoke tests
  ├─► manual approval
  └─► deploy prod (same template, prod.bicepparam)
```

```yaml
# GitHub Actions (OIDC, no secrets)
permissions: { id-token: write, contents: read }
steps:
  - uses: actions/checkout@v4
  - uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  - run: az bicep build --file infra/main.bicep
  - run: |
      az deployment sub what-if -l centralindia \
        -f infra/main.bicep -p infra/params/dev.bicepparam
  - run: |
      az deployment sub create -l centralindia -n "app-${{ github.run_number }}" \
        -f infra/main.bicep -p infra/params/dev.bicepparam
```

Azure DevOps: `AzureCLI@2` task with a workload-identity-federation service connection, same commands.

---

## 8. Gotchas

1. **Incremental mode never deletes.** Remove a resource from the template → it stays in Azure (and keeps billing). Use **deployment stacks** (`--action-on-unmanage deleteResources`) or delete explicitly.
2. **Complete mode is a foot-gun** — deletes everything in the RG not in the template, including things other teams added. Prefer stacks.
3. **Array properties are replaced wholesale.** Inline `subnets`, NSG `securityRules`, app `appSettings` → manual portal additions get wiped on redeploy. Child-resource subnets have the opposite problem (VNet redeploy can try to remove them). Pick inline subnets and own the whole VNet in IaC.
4. **`appSettings` in `siteConfig` replaces ALL settings** — settings added by other pipelines/portal disappear. Manage them all in one place (or use `Microsoft.Web/sites/config` `appsettings` resource, which also replaces).
5. **what-if noise** — shows changes for properties ARM normalizes (casing, defaults, read-only props). Learn to read past it; it's a preview, not a guarantee like Terraform plan.
6. **Role assignment names must be GUIDs** and deterministic → `guid(scope.id, principalId, roleId)`. Random GUID = `RoleAssignmentExists` on redeploy.
7. **`PrincipalNotFound`** right after creating a managed identity — Entra replication lag. Set `principalType: 'ServicePrincipal'` on the role assignment.
8. **Soft delete name conflicts** — Key Vault, Cognitive Services/OpenAI, API Management, App Configuration keep deleted names for days. Recreating with the same name fails until purged/recovered.
9. **Module `name` = nested deployment name.** Must be unique within the scope; looped modules need the index in the name. Max 64 chars.
10. **800 deployments per RG** history limit — old ones are auto-deleted, but naming every deployment with a timestamp makes history readable.
11. **`@secure()` param ≠ secret storage.** Don't `output` it; don't commit it in `.bicepparam`. Use `getSecret()` / `az.getSecret()` in `.bicepparam`, or pipeline secrets.
12. **`listKeys()` in outputs leaks keys** into deployment history. Prefer managed identity + RBAC (`disableLocalAuth: true`).
13. **Linux App Service plan needs `reserved: true`** — `kind: 'linux'` alone is not enough.
14. **Resource name rules differ per type** (storage: 3–24 lowercase alnum, global; Key Vault: 3–24, global; etc.). Compile succeeds, deploy fails.
15. **API versions**: `-preview` versions can change/disappear. Use GA unless you need the feature.
16. **Evaluated where?** Some expressions are compile-time (types, `loadTextContent`), others deploy-time (`reference()`, `uniqueString()` output). Things like `resource name` must be computable at the start of deployment — can't depend on another resource's runtime property (`BCP120`).
17. **Registry modules need egress** to `mcr.microsoft.com` (public) or your ACR from build agents. Self-hosted agents behind firewalls fail with `BCP192`.
18. **Policy denies show up at deploy time**, not compile. `az deployment group validate` catches most.

---

## 9. Production scenario FAQ

**Q1. A teammate removed a storage account from `main.bicep`; after deployment it's still there and still billed. Why and how do you prevent it?**
Default incremental mode only creates/updates. Either delete it manually, or move to **deployment stacks** with `--action-on-unmanage deleteResources` (or `detachAll` for safety in prod) so the stack knows what it owns.

**Q2. Prod deployment wiped app settings that the app team set in the portal.**
`siteConfig.appSettings` is a full replace. Decide one owner: IaC owns all settings (app team PRs into the param file) or IaC doesn't touch them at all. For secrets, use Key Vault references so values don't live in IaC.

**Q3. Pipeline fails with `RoleAssignmentExists` on the second run.**
The assignment `name` isn't deterministic. Use `guid(resource.id, principalId, roleDefinitionId)`. If an assignment was created manually with a random GUID, delete it once, then let Bicep own it.

**Q4. `what-if` says "Modify" on 20 properties every run but nothing actually changes.**
Normalization noise (defaults, casing, read-only values). Use `--result-format ResourceIdOnly` for a summary, or exclude known noise with `--exclude-change-types NoChange Ignore`. Track real drift with Azure Policy/Resource Graph rather than what-if.

**Q5. How do you deploy the same infra to dev/qa/prod?**
One template, one `.bicepparam` per env, subscription-scope `main.bicep` creating the env RG(s). Promote by running the same commit through env stages. Use types/`@allowed` so a bad env value fails at compile.

**Q6. Model deployment for Azure OpenAI fails with `InsufficientQuota` in prod but worked in dev.**
Quota (TPM) is per subscription + region + model + deployment type. Prod subscription has separate quota. Request quota increase or use a GlobalStandard / Data Zone deployment, and make capacity a param per env.

**Q7. Redeploying after a RG delete fails for Key Vault / OpenAI with "name already exists".**
Soft-deleted. `az keyvault recover` / `az keyvault purge`, `az cognitiveservices account purge`. In templates, use names with `uniqueString(resourceGroup().id)` and treat purge protection as intentional in prod.

**Q8. You need to bring existing manually-created resources under Bicep.**
Portal → Export template (or `az group export`) → `az bicep decompile` → clean up (remove read-only props, params for names) → what-if must show **no change** → then deploy. Alternatively the VS Code "Insert resource" command pulls a single resource's current definition.

**Q9. Should we write our own modules or use AVM?**
AVM for standard resources (Key Vault, storage, VNet) — they include diagnostics, private endpoints, RBAC, locks with tested defaults. Wrap them in thin internal modules with your org's naming/tags. Pin versions; mirror to your own ACR if agents lack internet.

**Q10. How do you stop people from deleting prod resources managed by Bicep?**
Deployment stack `--deny-settings-mode denyDelete` (with `--deny-settings-excluded-principals` for the pipeline identity), plus `Microsoft.Authorization/locks` (`CanNotDelete`) on critical RGs, plus least-privilege RBAC (humans Reader in prod).

**Q11. Bicep or Terraform for a new Azure client project?**
Follow the client's standard first. Otherwise: Azure-only + no state management wanted + new features needed fast → Bicep. Multi-cloud, existing TF skills/modules, strong plan/drift workflows, SaaS providers (GitHub, Datadog, Cloudflare) → Terraform.

---

## 10. Command reference

```bash
az bicep install | upgrade | version
az bicep build --file main.bicep [--stdout]
az bicep build-params --file dev.bicepparam
az bicep decompile --file template.json
az bicep lint --file main.bicep
az bicep format --file main.bicep
az bicep restore --file main.bicep            # pull registry modules
az bicep publish --file mod.bicep --target br:myacr.azurecr.io/bicep/mod:1.0.0

az deployment group validate -g RG -f main.bicep -p dev.bicepparam
az deployment group what-if  -g RG -f main.bicep -p dev.bicepparam
az deployment group create   -g RG -n deploy-$(date +%s) -f main.bicep -p dev.bicepparam
az deployment group show     -g RG -n NAME --query properties.outputs
az deployment operation group list -g RG -n NAME -o table      # which resource failed

az stack group create -n STACK -g RG -f main.bicep -p dev.bicepparam \
  --action-on-unmanage detachAll|deleteResources|deleteAll \
  --deny-settings-mode none|denyDelete|denyWriteAndDelete
az stack group list -g RG -o table
az stack group delete -n STACK -g RG --action-on-unmanage deleteAll

az ts create -n mytemplate -g RG -v 1.0 -f main.bicep          # template spec
```
