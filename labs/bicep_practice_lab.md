# Bicep — Hands-On Practice Lab

> 10 progressive exercises: first storage account → modules → subscription scope → messaging → AI stack → typed, reusable templates.
> Every template in this lab was compiled with Bicep CLI 0.48 (`bicep build`) — zero errors. The only line not compiled offline is the AVM registry module in Exercise 10.
> Companion notes: [`../bicep/bicep_notes.md`](../bicep/bicep_notes.md)

---

## 0. Setup (do once)

```bash
# 1. Tools
az version                       # Azure CLI ≥ 2.60
az bicep install                 # or: az bicep upgrade
az bicep version
code --install-extension ms-azuretools.vscode-bicep   # IntelliSense, linter, visualizer

# 2. Login + pick a SANDBOX subscription (never prod for labs)
az login
az account set --subscription "<sandbox-subscription-id>"
az account show -o table

# 3. Lab resource group
LOC=centralindia
RG=rg-bicep-lab
az group create -n $RG -l $LOC

# 4. Folder layout
mkdir -p bicep-lab/{ex01,ex02,ex03,ex04,ex05,ex06/modules,ex07,ex08,ex09,ex10}
```

The deployment loop you will repeat in every exercise:

```
  write .bicep ──► bicep build / lint ──► what-if ──► deploy ──► verify ──► change ──► what-if again
       ▲                                                                                   │
       └───────────────────────────────────────────────────────────────────────────────────┘
```

```bash
az bicep build --file main.bicep                       # compile → main.json (catches errors early)
az deployment group what-if   -g $RG -f main.bicep     # dry run: + create, ~ modify, - delete, = no change
az deployment group create    -g $RG -f main.bicep     # real deployment
az deployment group list      -g $RG -o table          # deployment history
```

> Cost tip: everything here is low-cost, but **delete the RG at the end of each day** (Section 11). AI Search Basic and App Service B1 bill hourly even when idle.

---

## Exercise 1 — First resource: Storage account

**Goal:** params, variables, `uniqueString()`, outputs.

`ex01/main.bicep`

```bicep
// Exercise 1 — first resource: a storage account
@description('Azure region for all resources')
param location string = resourceGroup().location

@description('Short prefix used in names (3-11 lowercase chars)')
@minLength(3)
@maxLength(11)
param prefix string = 'cvlab'

// uniqueString() is deterministic per RG -> same name on every redeploy
var storageName = toLower('${prefix}${uniqueString(resourceGroup().id)}')

resource stg 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}

output storageAccountName string = stg.name
output blobEndpoint string = stg.properties.primaryEndpoints.blob
```

```bash
cd ex01
az deployment group what-if -g $RG -f main.bicep
az deployment group create  -g $RG -f main.bicep --query properties.outputs
az storage account list -g $RG -o table
```

**Try this:**

1. Deploy a second time without changes → what-if shows `=` / no change. That's **idempotency**.
2. Change `allowBlobPublicAccess` to `true` → what-if shows `~ Modify`. Revert it.
3. Pass `prefix=ab` → deployment fails *before* calling Azure. Which decorator caught it?
4. Open `main.json` after `az bicep build` — find what `uniqueString()` became (it stays an ARM expression, evaluated by Azure, not by your laptop).

---

## Exercise 2 — Parameter files & env-based config

**Goal:** `@allowed`, config maps, `union()`, child resources with `parent`, `.bicepparam` files.

`ex02/main.bicep`

```bicep
// Exercise 2 — params, decorators, env-driven config
@allowed([
  'dev'
  'prod'
])
param env string

param location string = resourceGroup().location

@description('Tags applied to every resource')
param tags object = {}

var envConfig = {
  dev: {
    skuName: 'Standard_LRS'
    retentionDays: 7
  }
  prod: {
    skuName: 'Standard_ZRS'
    retentionDays: 30
  }
}

var commonTags = union(tags, {
  env: env
  managedBy: 'bicep'
})

resource stg 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'st${env}${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: envConfig[env].skuName
  }
  kind: 'StorageV2'
  tags: commonTags
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

resource blobSvc 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: stg
  name: 'default'
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: envConfig[env].retentionDays
    }
  }
}

output sku string = stg.sku.name
```

`ex02/dev.bicepparam`

```bicep
using './main.bicep'

param env = 'dev'
param tags = {
  owner: 'chaitanya'
  costCenter: 'training'
}
```

`ex02/prod.bicepparam`

```bicep
using './main.bicep'

param env = 'prod'
param tags = {
  owner: 'chaitanya'
  costCenter: 'training'
}
```

```bash
cd ../ex02
az deployment group create -g $RG -p dev.bicepparam   # template comes from the `using` line
az deployment group what-if -g $RG -p prod.bicepparam
```

**Try this:**

1. Note you didn't pass `-f` — the `.bicepparam` file's `using` line points to the template.
2. Override one value on the CLI: `az deployment group create -g $RG -p dev.bicepparam -p env=prod` — which wins?
3. Add `'qa'` to `@allowed` but **not** to `envConfig` → deploy with env=qa. Compile passes, deployment fails. Why? (Hint: lookups on a map happen at deploy time.) Fix it with safe access + default: `envConfig[?env].?skuName ?? 'Standard_LRS'`.

---

## Exercise 3 — Loops & conditions: VNet + NSG

**Goal:** `for` loops, `if` conditions, ternary, inline child resources.

`ex03/main.bicep`

```bicep
// Exercise 3 — loops + conditions: VNet, subnets, optional NSG
param location string = resourceGroup().location
param vnetName string = 'vnet-lab'
param addressPrefix string = '10.10.0.0/16'
param deployNsg bool = true

param subnets array = [
  {
    name: 'snet-app'
    prefix: '10.10.1.0/24'
  }
  {
    name: 'snet-data'
    prefix: '10.10.2.0/24'
  }
  {
    name: 'snet-pe'
    prefix: '10.10.3.0/24'
  }
]

resource nsg 'Microsoft.Network/networkSecurityGroups@2024-05-01' = if (deployNsg) {
  name: 'nsg-lab'
  location: location
  properties: {
    securityRules: [
      {
        name: 'Allow-HTTPS-Inbound'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '443'
        }
      }
    ]
  }
}

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
    // subnets defined INLINE (not as child resources) to avoid redeploy wiping them
    subnets: [for s in subnets: {
      name: s.name
      properties: {
        addressPrefix: s.prefix
        networkSecurityGroup: deployNsg ? {
          id: nsg.id
        } : null
      }
    }]
  }
}

output subnetIds array = [for (s, i) in subnets: vnet.properties.subnets[i].id]
```

```bash
cd ../ex03
az deployment group create -g $RG -f main.bicep
az network vnet subnet list -g $RG --vnet-name vnet-lab -o table
az deployment group create -g $RG -f main.bicep -p deployNsg=false
```

**Try this (important gotcha):**

1. Add a subnet manually in the portal (`snet-manual`, `10.10.9.0/24`). Redeploy the template. What happened to `snet-manual`?
   → It is **removed**, because `subnets` is an array property — Bicep sends the whole array. **Lesson:** never mix portal changes and IaC on the same resource.
2. Now try the "child resource" style (`Microsoft.Network/virtualNetworks/subnets` as separate resources) and redeploy twice. Read the notes on why inline subnets are preferred.
3. Use `@batchSize(1)` on a loop and observe deployment order in the portal (Deployments blade).

---

## Exercise 4 — Key Vault, secrets, RBAC, `existing`

**Goal:** `@secure()`, role assignments with `guid()`, `existing` keyword, `getSecret()`.

`ex04/main.bicep`

```bicep
// Exercise 4 — Key Vault (RBAC mode), existing resources, role assignment with guid()
param location string = resourceGroup().location

@description('Object ID of the user/group/SP that should read secrets (az ad signed-in-user show --query id -o tsv)')
param readerPrincipalId string

@secure()
@description('Demo secret value — never put real secrets in param files committed to git')
param demoSecretValue string

// Built-in role: Key Vault Secrets User
var kvSecretsUserRoleId = '4633458b-17de-408a-b874-0445c86b69e6'

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-lab-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
    enablePurgeProtection: true
  }
}

resource secret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: kv
  name: 'demo-secret'
  properties: {
    value: demoSecretValue
  }
}

resource kvRead 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(kv.id, readerPrincipalId, kvSecretsUserRoleId)  // deterministic -> idempotent
  scope: kv
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', kvSecretsUserRoleId)
    principalId: readerPrincipalId
  }
}

output keyVaultName string = kv.name
output secretUri string = secret.properties.secretUri
```

```bash
cd ../ex04
ME=$(az ad signed-in-user show --query id -o tsv)
read -s SECRET    # type something; it won't echo
az deployment group create -g $RG -f main.bicep \
  -p readerPrincipalId=$ME demoSecretValue="$SECRET"

KV=$(az keyvault list -g $RG --query [0].name -o tsv)
az keyvault secret show --vault-name $KV -n demo-secret --query value -o tsv
```

Now reference the vault as an **existing** resource and pass the secret into a module:

`ex04/reference-existing.bicep`

```bicep
// Exercise 4b — reference an EXISTING vault and pass a secret to a module param
param keyVaultName string

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: keyVaultName
}

module consumer 'consumer.bicep' = {
  name: 'consumer'
  params: {
    adminPassword: kv.getSecret('demo-secret')   // only allowed for @secure() module params
  }
}
```

`ex04/consumer.bicep`

```bicep
// In a real template this would be a VM/SQL server adminPassword property.
// Never output a @secure() value — it ends up in deployment history in plain text.
@secure()
param adminPassword string

param location string = resourceGroup().location

resource sql 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'sql-lab-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    administratorLogin: 'sqladminlab'
    administratorLoginPassword: adminPassword
    minimalTlsVersion: '1.2'
  }
}
```

```bash
az deployment group what-if -g $RG -f reference-existing.bicep -p keyVaultName=$KV
```

**Try this:**

1. Redeploy `main.bicep` — the role assignment shows no change because `guid()` gives the same name. Replace `guid(...)` with `newGuid()` (only allowed as a param default) and see the "RoleAssignmentExists" error on redeploy.
2. Add `output s string = demoSecretValue` → the linter warns (`outputs-should-not-contain-secrets`). Check deployment history in the portal to see why it matters.
3. Delete the RG, recreate it, redeploy → `VaultAlreadyExists`/soft-delete conflict. Recover with `az keyvault recover` or use a new name. (Purge protection = you **can't** purge for N days.)

---

## Exercise 5 — App Service with managed identity

**Goal:** implicit dependencies, `identity`, Linux plan gotchas.

`ex05/main.bicep`

```bicep
// Exercise 5 — App Service + managed identity + implicit dependencies
param location string = resourceGroup().location
param appName string = 'app-lab-${uniqueString(resourceGroup().id)}'

@allowed([
  'B1'
  'P0v3'
])
param planSku string = 'B1'

resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-lab'
  location: location
  sku: {
    name: planSku
  }
  kind: 'linux'
  properties: {
    reserved: true          // required for Linux plans
  }
}

resource ai 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-lab'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

resource app 'Microsoft.Web/sites@2023-12-01' = {
  name: appName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: plan.id                      // implicit dependsOn plan
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: ai.properties.ConnectionString  // implicit dependsOn ai
        }
      ]
    }
  }
}

output defaultHostName string = app.properties.defaultHostName
output principalId string = app.identity.principalId
```

```bash
cd ../ex05
az deployment group create -g $RG -f main.bicep --query properties.outputs
```

**Try this:**

1. Remove `reserved: true` → the plan is created as **Windows** even though `kind: 'linux'`. Classic gotcha.
2. Use the `principalId` output to grant the web app **Key Vault Secrets User** on the vault from Exercise 4 (add a role assignment resource referencing the vault with `existing`).
3. Add an app setting `MY_SECRET` = `@Microsoft.KeyVault(SecretUri=<secretUri>)` (Key Vault reference) and confirm in the portal it resolves (green check).
4. Open the VS Code Bicep visualizer (`Ctrl+Shift+P → Bicep: Open Visualizer`) — see the implicit dependency arrows.

---

## Exercise 6 — Modules: network + storage + private endpoint

**Goal:** split into modules, pass outputs between modules, private endpoint.

```
 main.bicep
   ├── module network  ──outputs.peSubnetId──┐
   ├── module storage  ──outputs.id──────────┤
   └── module pe  ◄──────────────────────────┘   (waits for both automatically)
```

`ex06/modules/storage.bicep`

```bicep
param name string
param location string
param skuName string = 'Standard_LRS'
param tags object = {}

resource stg 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: name
  location: location
  sku: {
    name: skuName
  }
  kind: 'StorageV2'
  tags: tags
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    publicNetworkAccess: 'Disabled'
  }
}

output id string = stg.id
output name string = stg.name
```

`ex06/modules/network.bicep`

```bicep
param name string
param location string
param addressPrefix string
param peSubnetPrefix string

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: name
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
    subnets: [
      {
        name: 'snet-pe'
        properties: {
          addressPrefix: peSubnetPrefix
        }
      }
    ]
  }
}

output peSubnetId string = vnet.properties.subnets[0].id
output vnetId string = vnet.id
```

`ex06/modules/private-endpoint.bicep`

```bicep
param name string
param location string
param subnetId string
param targetResourceId string
param groupId string

resource pe 'Microsoft.Network/privateEndpoints@2024-05-01' = {
  name: name
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${name}-conn'
        properties: {
          privateLinkServiceId: targetResourceId
          groupIds: [
            groupId
          ]
        }
      }
    ]
  }
}

output id string = pe.id
```

`ex06/main.bicep`

```bicep
// Exercise 6 — modules: compose network + storage + private endpoint
param location string = resourceGroup().location
param env string = 'dev'

var suffix = uniqueString(resourceGroup().id)
var tags = {
  env: env
  managedBy: 'bicep'
}

module network 'modules/network.bicep' = {
  name: 'network-${env}'
  params: {
    name: 'vnet-${env}'
    location: location
    addressPrefix: '10.20.0.0/16'
    peSubnetPrefix: '10.20.1.0/24'
  }
}

module storage 'modules/storage.bicep' = {
  name: 'storage-${env}'
  params: {
    name: 'st${env}${suffix}'
    location: location
    tags: tags
  }
}

// module outputs -> module params = implicit dependency on BOTH modules
module pe 'modules/private-endpoint.bicep' = {
  name: 'pe-blob-${env}'
  params: {
    name: 'pe-${storage.outputs.name}-blob'
    location: location
    subnetId: network.outputs.peSubnetId
    targetResourceId: storage.outputs.id
    groupId: 'blob'
  }
}

output storageId string = storage.outputs.id
```

```bash
cd ../ex06
az deployment group create -g $RG -f main.bicep
az deployment group list -g $RG -o table      # one deployment PER module (nested deployments)
```

**Try this:**

1. Each module creates a **nested deployment** named by `name:`. Deploy the same module twice in a loop with the same `name` → conflict. Fix with `'storage-${i}'`.
2. The private endpoint works but `nslookup <account>.blob.core.windows.net` from a VM in the VNet still returns a public IP. Why? Add a `privateDnsZones` module (`privatelink.blob.core.windows.net`), a VNet link, and a `privateDnsZoneGroups` child on the PE. (This is the #1 private endpoint mistake in production.)

---

## Exercise 7 — Subscription scope: create resource groups

**Goal:** `targetScope`, deploying modules into another scope.

`ex07/main.bicep`

```bicep
// Exercise 7 — subscription scope: create RGs and deploy into them
targetScope = 'subscription'

param location string = 'centralindia'
param envs array = [
  'dev'
  'qa'
]

resource rgs 'Microsoft.Resources/resourceGroups@2024-03-01' = [for e in envs: {
  name: 'rg-bicep-lab-${e}'
  location: location
  tags: {
    env: e
  }
}]

module stg '../ex06/modules/storage.bicep' = [for (e, i) in envs: {
  name: 'stg-${e}'
  scope: rgs[i]                      // deploy INTO the RG created above
  params: {
    name: 'st${e}${uniqueString(subscription().id, e)}'
    location: location
  }
}]

output rgNames array = [for (e, i) in envs: rgs[i].name]
```

```bash
cd ../ex07
az deployment sub what-if -l $LOC -f main.bicep
az deployment sub create  -l $LOC -f main.bicep
az group list --query "[?starts_with(name,'rg-bicep-lab-')].name" -o table
```

**Try this:**

1. Notice `az deployment sub` needs `-l` — that's where the **deployment metadata** is stored, not where resources go.
2. Remove `qa` from `envs` and redeploy. Was `rg-bicep-lab-qa` deleted? (No — incremental mode never deletes. See deployment stacks in Exercise 10.)
3. Clean up: `az group delete -n rg-bicep-lab-dev --yes --no-wait` (and qa).

---

## Exercise 8 — Messaging: Service Bus + Event Grid

**Goal:** child resources 3 levels deep, SQL filters, Event Grid system topic → Service Bus.

```
  [Storage: container "uploads"]
        │ BlobCreated (*.pdf only)
        ▼
  [Event Grid system topic] ── event subscription (filter + retry) ──► [SB queue: blob-events]

  [Your app] ──► [SB topic: order-events] ──► sub: email-service  (all messages)
                                         └──► sub: fraud-check    (amount > 50000)
```

`ex08/main.bicep`

```bicep
// Exercise 8 — messaging: Service Bus (queue + topic/subscription) fed by Event Grid
param location string = resourceGroup().location
var suffix = uniqueString(resourceGroup().id)

// ---------- Service Bus ----------
resource sb 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-lab-${suffix}'
  location: location
  sku: {
    name: 'Standard'          // topics need Standard or Premium (not Basic)
    tier: 'Standard'
  }
  properties: {
    minimumTlsVersion: '1.2'
    disableLocalAuth: true    // force Entra ID (no SAS keys)
  }
}

resource ordersQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: sb
  name: 'orders'
  properties: {
    lockDuration: 'PT1M'
    maxDeliveryCount: 10
    deadLetteringOnMessageExpiration: true
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
  }
}

resource blobEventsQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: sb
  name: 'blob-events'
  properties: {
    maxDeliveryCount: 5
  }
}

resource notifyTopic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: sb
  name: 'order-events'
}

resource emailSub 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: notifyTopic
  name: 'email-service'
  properties: {
    maxDeliveryCount: 10
  }
}

resource highValueSub 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: notifyTopic
  name: 'fraud-check'
}

resource highValueRule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: highValueSub
  name: 'amount-over-50k'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: 'amount > 50000'   // matches an application property on the message
    }
  }
}

// ---------- Event Grid: storage blob-created -> Service Bus queue ----------
resource stg 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'stevt${suffix}'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

resource sysTopic 'Microsoft.EventGrid/systemTopics@2024-06-01-preview' = {
  name: 'egst-${stg.name}'
  location: location
  properties: {
    source: stg.id
    topicType: 'Microsoft.Storage.StorageAccounts'
  }
}

resource blobCreatedSub 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2024-06-01-preview' = {
  parent: sysTopic
  name: 'blob-created-to-sb'
  properties: {
    eventDeliverySchema: 'CloudEventSchemaV1_0'
    destination: {
      endpointType: 'ServiceBusQueue'
      properties: {
        resourceId: blobEventsQueue.id
      }
    }
    filter: {
      includedEventTypes: [
        'Microsoft.Storage.BlobCreated'
      ]
      subjectBeginsWith: '/blobServices/default/containers/uploads/'
      subjectEndsWith: '.pdf'
    }
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440
    }
  }
}

output serviceBusFqdn string = '${sb.name}.servicebus.windows.net'
output storageName string = stg.name
```

```bash
cd ../ex08
az deployment group create -g $RG -f main.bicep
STG=$(az deployment group show -g $RG -n main --query properties.outputs.storageName.value -o tsv)
ME=$(az ad signed-in-user show --query id -o tsv)

# you need data-plane roles because local auth is disabled
az role assignment create --assignee $ME --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -n $STG -g $RG --query id -o tsv)

az storage container create --account-name $STG -n uploads --auth-mode login
echo hello > test.pdf
az storage blob upload --account-name $STG -c uploads -f test.pdf -n test.pdf --auth-mode login
# Portal → Service Bus namespace → Queues → blob-events → Service Bus Explorer → Peek
```

**Try this:**

1. Upload `test.txt` → no message (filtered by `subjectEndsWith`).
2. Grant yourself **Azure Service Bus Data Owner**, send a message to topic `order-events` from Service Bus Explorer with custom property `amount = 70000` → appears in both subscriptions. Send `amount = 100` → only `email-service`.
3. Receive a message in peek-lock mode and **abandon** it 10 times → it moves to the **dead-letter queue** (`maxDeliveryCount`).
4. Change the SKU to `Basic` → deployment fails on the topic. Why?

---

## Exercise 9 — AI stack: Foundry + model deployments + AI Search

**Goal:** deploy the infra behind a RAG app with **keyless (Entra ID) auth**.

```
 ┌───────────────────── Foundry resource (kind: AIServices) ───────────────────┐
 │  project: proj-lab           deployments: gpt-4.1-mini, text-embedding-3-small│
 └──────────▲──────────────────────────────────────────────▲───────────────────┘
            │ "Cognitive Services OpenAI User"              │ calls chat model
            │ (Search MI → embeddings)                      │
 ┌──────────┴──────────┐   "Search Index Data Reader"   ┌───┴──────────────┐
 │  Azure AI Search    │ ◄──────────────────────────────│ Project MI / app │
 └─────────────────────┘                                 └──────────────────┘
```

`ex09/main.bicep`

```bicep
// Exercise 9 — AI stack: Foundry (AIServices) account + project + model deployment + AI Search
param location string = resourceGroup().location

@description('Check availability first: az cognitiveservices model list -l <region> -o table')
param chatModelName string = 'gpt-4.1-mini'
param chatModelVersion string = '2025-04-14'
param embeddingModelName string = 'text-embedding-3-small'
param embeddingModelVersion string = '1'

@description('Capacity in units of 1K tokens-per-minute (TPM) for Standard deployments')
param chatCapacity int = 10

var suffix = uniqueString(resourceGroup().id)

// Foundry resource = Microsoft.CognitiveServices/accounts with kind AIServices
resource foundry 'Microsoft.CognitiveServices/accounts@2025-06-01' = {
  name: 'aif-lab-${suffix}'
  location: location
  kind: 'AIServices'
  sku: {
    name: 'S0'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    customSubDomainName: 'aif-lab-${suffix}'   // required for Entra ID auth
    allowProjectManagement: true               // enables Foundry projects under this resource
    disableLocalAuth: true                     // no API keys, Entra ID only
    publicNetworkAccess: 'Enabled'             // lab only; prod = Disabled + private endpoint
  }
}

resource project 'Microsoft.CognitiveServices/accounts/projects@2025-06-01' = {
  parent: foundry
  name: 'proj-lab'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {}
}

resource chat 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundry
  name: chatModelName
  sku: {
    name: 'GlobalStandard'
    capacity: chatCapacity
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: chatModelName
      version: chatModelVersion
    }
  }
}

// Deployments on the same account must be created one after another
resource embed 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundry
  name: embeddingModelName
  dependsOn: [
    chat
  ]
  sku: {
    name: 'Standard'
    capacity: 10
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: embeddingModelName
      version: embeddingModelVersion
    }
  }
}

resource search 'Microsoft.Search/searchServices@2024-06-01-preview' = {
  name: 'srch-lab-${suffix}'
  location: location
  sku: {
    name: 'basic'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    replicaCount: 1
    partitionCount: 1
    hostingMode: 'default'
    semanticSearch: 'free'
    authOptions: {
      aadOrApiKey: {
        aadAuthFailureMode: 'http401WithBearerChallenge'
      }
    }
  }
}

// Search needs to call the embedding model (integrated vectorization)
var cogServicesOpenAIUser = '5e0bd9bd-7b93-4f28-af87-19fc36ad61bd'
resource searchToOpenAI 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(foundry.id, search.id, cogServicesOpenAIUser)
  scope: foundry
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', cogServicesOpenAIUser)
    principalId: search.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Foundry project needs to read the search index (RAG / agents)
var searchIndexDataReader = '1407120a-92aa-4202-b7e9-c0e197c71c8f'
resource projectToSearch 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(search.id, project.id, searchIndexDataReader)
  scope: search
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', searchIndexDataReader)
    principalId: project.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

output foundryEndpoint string = foundry.properties.endpoint
output searchEndpoint string = 'https://${search.name}.search.windows.net'
```

```bash
cd ../ex09
# 1. Check model + quota availability in your region FIRST
az cognitiveservices model list -l $LOC \
  --query "[?model.name=='gpt-4.1-mini'].{v:model.version, sku:model.skus[0].name}" -o table
az cognitiveservices usage list -l $LOC -o table | grep -i gpt

# 2. Deploy (try eastus2 / swedencentral if your region lacks the model)
az deployment group create -g $RG -f main.bicep --query properties.outputs

# 3. Give yourself rights to call the model
ME=$(az ad signed-in-user show --query id -o tsv)
AIF=$(az cognitiveservices account list -g $RG --query [0].id -o tsv)
az role assignment create --assignee $ME --role "Cognitive Services OpenAI User" --scope $AIF

# 4. Call the model with an Entra token (no API key)
EP=$(az cognitiveservices account show --ids $AIF --query properties.endpoint -o tsv)
TOKEN=$(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)
curl -s "${EP}openai/v1/chat/completions" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"gpt-4.1-mini","messages":[{"role":"user","content":"Explain Bicep in one line"}]}' | jq -r '.choices[0].message.content'
```

> If the `/openai/v1/` path returns 404 in your region/API version, use the classic path:
> `${EP}openai/deployments/gpt-4.1-mini/chat/completions?api-version=2024-10-21`.

**Try this:**

1. Set `chatCapacity` higher than your quota → `InsufficientQuota`. Read the error; check quota in the Foundry portal (Management → Quota).
2. Remove the `dependsOn: [chat]` on the embedding deployment and deploy both fresh → you may hit a conflict error ("another operation is in progress"). That's why deployments are serialized.
3. In the Foundry portal open the project → Playground → chat with your deployment → "Add your data" → point it at AI Search (needs an index; use the "Import and vectorize data" wizard on a blob container).
4. Delete the RG and immediately redeploy → `FlagMustBeSetForRestore` / name conflict: Cognitive Services accounts are **soft-deleted**. Purge with `az cognitiveservices account purge`.

---

## Exercise 10 — Types, functions, AVM, deployment stacks

**Goal:** make templates safe to reuse; manage lifecycle (including deletes).

`ex10/types.bicep`

```bicep
// Shared user-defined types — import them elsewhere with: import { appConfig } from 'types.bicep'
@export()
type environment = 'dev' | 'qa' | 'prod'

@export()
type subnetSpec = {
  name: string
  @description('CIDR, e.g. 10.0.1.0/24')
  prefix: string
  delegation: string?          // optional property
}

@export()
type appConfig = {
  env: environment
  location: string
  subnets: subnetSpec[]
}

@export()
func resourceName(kind string, env environment, region string) string => '${kind}-${env}-${region}'
```

`ex10/main.bicep`

```bicep
// Exercise 10 — user-defined types, functions, AVM registry modules
import { appConfig, resourceName } from 'types.bicep'

param config appConfig

resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: resourceName('vnet', config.env, 'cin')
  location: config.location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.30.0.0/16'
      ]
    }
    subnets: [for s in config.subnets: {
      name: s.name
      properties: {
        addressPrefix: s.prefix
        delegations: s.?delegation == null ? [] : [
          {
            name: 'delegation'
            properties: {
              serviceName: s.delegation!
            }
          }
        ]
      }
    }]
  }
}

// Azure Verified Module from the public Bicep registry (check latest version on aka.ms/avm)
module kv 'br/public:avm/res/key-vault/vault:0.13.0' = {
  name: 'kv-avm'
  params: {
    name: 'kv-${config.env}-${uniqueString(resourceGroup().id)}'
    location: config.location
    enableRbacAuthorization: true
    enablePurgeProtection: config.env == 'prod'
  }
}
```

`ex10/dev.bicepparam`

```bicep
using './main.bicep'

param config = {
  env: 'dev'
  location: 'centralindia'
  subnets: [
    { name: 'snet-app', prefix: '10.30.1.0/24' }
    { name: 'snet-func', prefix: '10.30.2.0/24', delegation: 'Microsoft.Web/serverFarms' }
  ]
}
```

```bash
cd ../ex10
az bicep restore --file main.bicep         # pulls AVM module from mcr.microsoft.com
az deployment group what-if -g $RG -p dev.bicepparam

# Deployment STACK = deployment that remembers what it manages and can delete/lock it
az stack group create -n lab-stack -g $RG -p dev.bicepparam \
  --action-on-unmanage deleteResources \
  --deny-settings-mode denyDelete
az stack group show -n lab-stack -g $RG --query "resources[].id" -o tsv
```

**Try this:**

1. Put `env: 'stage'` in the param file → **compile-time** error (type checking). Compare with Exercise 2 where a bad lookup only failed at deploy time.
2. Try deleting the Key Vault in the portal → blocked by the stack's deny assignment.
3. Remove the VNet from `main.bicep`, run `az stack group create` again → the VNet is **deleted** (`deleteResources`). Normal deployments never do this.
4. Find the latest AVM version: browse `https://aka.ms/avm` → Bicep → `avm/res/key-vault/vault`. Pin it; never use an unpinned version.

---

## 11. Cleanup

```bash
az stack group delete -n lab-stack -g $RG --action-on-unmanage deleteAll --yes
az group delete -n $RG --yes --no-wait
az group list --query "[?starts_with(name,'rg-bicep-lab')].name" -o tsv | xargs -r -n1 az group delete --yes --no-wait -n

# Soft-deleted leftovers that block name reuse:
az keyvault list-deleted -o table
az cognitiveservices account list-deleted -o table
```

---

## 12. Break-fix challenges (interview-style)

| # | Symptom | Where to look | Root cause / fix |
|---|---------|---------------|------------------|
| 1 | `InvalidTemplateDeployment` with `StorageAccountAlreadyTaken` | name | Storage names are global; use `uniqueString()` |
| 2 | Subnet disappeared after redeploy | Ex 3 | Inline `subnets` array overwrites portal changes |
| 3 | `RoleAssignmentExists` on redeploy | Ex 4 | Assignment name not deterministic — use `guid(scope, principal, role)` |
| 4 | `PrincipalNotFound` on fresh deploy | Ex 4/9 | Entra replication lag for new managed identity — set `principalType: 'ServicePrincipal'` |
| 5 | Linux web app created as Windows | Ex 5 | Missing `reserved: true` on plan |
| 6 | PE deployed but app still resolves public IP | Ex 6 | Missing private DNS zone + zone group + VNet link |
| 7 | Removed resource from template, still exists | Ex 7 | Incremental mode — use deployment stacks or delete explicitly |
| 8 | Topic creation fails | Ex 8 | Service Bus Basic tier has no topics |
| 9 | `InsufficientQuota` on model deployment | Ex 9 | TPM quota per region/model/subscription — lower capacity or request quota |
| 10 | `FlagMustBeSetForRestore` | Ex 9 | Soft-deleted Cognitive Services account with same name — purge or `restore: true` |
| 11 | `BCP192 Unable to restore artifact` in CI | Ex 10 | Agent has no egress to `mcr.microsoft.com` — allowlist or vendor the module |

---

[🏠 Back to Labs](README.md)
