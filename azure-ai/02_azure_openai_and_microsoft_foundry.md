# Azure OpenAI & Microsoft Foundry — DevOps Notes

> Prerequisite: [`01_ai_terms_for_beginners.md`](01_ai_terms_for_beginners.md). IaC: Bicep lab Exercise 9.
> Naming history (you'll see all of these in docs/blogs):
> **Azure AI Studio** (2023) → **Azure AI Foundry** (Nov 2024) → **Microsoft Foundry** (Nov 2025).
> The older **hub-based** setup now shows up in docs as **"Foundry (classic)"**. New work should use **Foundry projects** on a Foundry resource.

---

## 1. How the pieces relate

```
 Azure subscription
  └── Resource group
       └── Foundry resource   (ARM: Microsoft.CognitiveServices/accounts, kind = "AIServices")
            │   endpoint: https://<name>.cognitiveservices.azure.com / .openai.azure.com / .services.ai.azure.com
            │   networking, keys/Entra auth, CMK, quotas live HERE
            │
            ├── Model deployments  (accounts/deployments)
            │     ├── gpt-4.1-mini   (GlobalStandard, 50K TPM)
            │     └── text-embedding-3-small (Standard, 100K TPM)
            │
            └── Projects  (accounts/projects)  — workspace per team/app
                  ├── Agents (Foundry Agent Service)
                  ├── Connections (AI Search, Storage, Cosmos DB, Bing grounding...)
                  ├── Evaluations, traces (→ Application Insights)
                  └── Files, vector stores, threads
```

**Azure OpenAI** = the OpenAI models (GPT, o-series, embeddings, image, audio) running in Azure's datacenters with Azure security/compliance. Today it is consumed **through a Foundry resource** (a standalone `kind: OpenAI` resource still exists and still works, but Foundry is where new features land).

**Microsoft Foundry** = the platform/portal (ai.azure.com) + APIs to pick models, deploy them, build agents, evaluate, monitor and govern.

### Classic vs new

| | Foundry (classic, hub-based) | Foundry (new, project on Foundry resource) |
|---|---|---|
| Top resource | **Hub** = `Microsoft.MachineLearningServices/workspaces` (kind Hub) | **Foundry resource** = `Microsoft.CognitiveServices/accounts` (kind AIServices) |
| Project | ML workspace (kind Project) | `accounts/projects` child resource |
| Dependencies | Storage + Key Vault (+ACR, AppInsights) required | Fewer required dependencies |
| Use when | Existing setups, some Azure ML features (prompt flow, managed compute) | All new GenAI/agent work |

---

## 2. Key terms

| Term | Meaning |
|------|---------|
| **Model catalog** | List of models: OpenAI, Microsoft (Phi), Meta Llama, Mistral, DeepSeek, xAI, Cohere, etc. |
| **Models sold directly by Azure** | Billed and supported by Microsoft (Azure OpenAI + some partner models). |
| **Models from partners/community** | Billed via Marketplace or deployed on your compute. |
| **Deployment** | A named instance of a model + version + SKU + capacity. **Your code calls the deployment name**, not the model name. |
| **Model version** | e.g. `gpt-4.1-mini` `2025-04-14`. Versions get retired. |
| **Version upgrade policy** | Auto-update to default version / when expired / no auto-upgrade. |
| **TPM / RPM quota** | Per subscription × region × model × deployment type. Capacity unit for Standard ≈ 1K TPM. |
| **PTU (Provisioned Throughput Unit)** | Reserved model capacity: predictable latency, fixed hourly/monthly cost (reservations available). |
| **Spillover** | Overflow from a provisioned deployment to a standard deployment when PTU is full. |
| **Batch** | Async jobs (JSONL file in, results within ~24h) at ~50% cost. |
| **Content filter / guardrails** | Configurable safety policy attached to a deployment (and agents). |
| **Prompt Shields** | Detect jailbreak / indirect prompt injection. |
| **Responses API** | Newer OpenAI API (stateful conversations, tools, built-in tool calling). |
| **Chat Completions API** | Classic messages-in, message-out API. Still widely used. |
| **v1 API** | Azure's newer `/openai/v1/` path compatible with the standard OpenAI SDK, without monthly `api-version`. |
| **Foundry Agent Service** | Hosted agents: model + instructions + tools (file search, code interpreter, AI Search, Bing, OpenAPI, MCP, functions), with threads/state managed for you. |
| **Connections** | Stored links from a project to other resources (AI Search, Storage, etc.), ideally using managed identity. |
| **Evaluations** | Built-in evaluators (groundedness, relevance, coherence, safety) run on datasets or live traffic. |
| **Tracing** | OpenTelemetry traces of prompts/tool calls → Application Insights. |
| **Foundry Local** | Run models on a local device. |
| **AI gateway** | APIM policies for token limits, load balancing, semantic caching, usage metrics. |

---

## 3. Deployment types (data residency × billing)

| Type | Where processed | Billing | Typical use |
|------|-----------------|---------|-------------|
| **Global Standard** | Any Azure region worldwide | Pay per token | Default; highest quota, newest models first |
| **Data Zone Standard** | Within a data zone (US or EU) | Pay per token | EU/US residency needs |
| **Standard (regional)** | That region only | Pay per token | Strict regional residency (fewer models) |
| **Global / Data Zone / Regional Provisioned** | As above | PTU (reserved) | Steady high-volume production |
| **Global / Data Zone Batch** | As above | ~50% off, async | Bulk processing, evaluations, nightly jobs |
| **Developer** | Global | Low cost, no SLA | Testing fine-tuned models |

> Data **at rest** (files, fine-tuned models, threads) stays in the resource's region/geo. The deployment type controls where **inference processing** may happen. Check with compliance for clients in regulated sectors (e.g. India data localization requirements).

```
 Decision:
   Need strict in-region processing?  ── yes ──► Standard (regional) / Regional Provisioned
           │ no
   EU/US zone enough?                 ── yes ──► Data Zone Standard / Provisioned
           │ no
   Steady, high, predictable load?    ── yes ──► Global Provisioned (PTU) + spillover
           │ no
   Offline / bulk?                    ── yes ──► Global Batch
           │ no
           └──────────────────────────────────► Global Standard
```

---

## 4. Production architecture (reference)

```
                         ┌──────────── Hub VNet ─────────────┐
 Users ─► Front Door/WAF ─► App Gateway ─►  APIM (AI gateway)│
                         │   • Entra ID / JWT validation      │
                         │   • llm-token-limit per team       │
                         │   • load balance backends + retry  │
                         │   • llm-emit-token-metric          │
                         └───────┬───────────────┬───────────┘
                     private endpoint      private endpoint
                                 ▼               ▼
              ┌──────────────────────┐  ┌──────────────────────┐
              │ Foundry resource     │  │ Foundry resource     │
              │ centralindia (PTU)   │  │ swedencentral (Std)  │   ← failover / spillover
              └─────────▲────────────┘  └──────────────────────┘
                        │ managed identity (Cognitive Services OpenAI User)
 ┌──────────────┐       │                ┌─────────────────┐
 │ App on AKS / │───────┘                │ Azure AI Search │ (private endpoint)
 │ App Service  │───────────────────────►│ (RAG index)     │
 └──────┬───────┘                        └─────────────────┘
        │ OpenTelemetry
        ▼
   Application Insights / Log Analytics  (tokens, latency, 429s, traces)
```

Checklist:

- `disableLocalAuth: true` → **Entra ID only**, no API keys. Apps use managed identity with **Cognitive Services OpenAI User** (call models) — not Contributor.
- `publicNetworkAccess: Disabled` + private endpoint + private DNS zones (`privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`, `privatelink.services.ai.azure.com`).
- Agent Service with your own VNet ("network-secured / BYO VNet" setup) needs subnet delegation and its own dependencies (Storage, Cosmos DB, AI Search) — plan subnets early.
- Diagnostic settings → Log Analytics. Alerts on 429 rate, latency P95, token usage, content-filter blocks.
- Pin model versions in prod; set upgrade policy deliberately; track retirement dates.
- Budget alerts per resource group; per-team token quotas via APIM.

---

## 5. Calling a model (keyless)

```python
# pip install openai azure-identity
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")

client = OpenAI(
    base_url="https://<resource>.openai.azure.com/openai/v1/",
    api_key=token_provider,            # token, not a key
)

resp = client.chat.completions.create(
    model="gpt-4.1-mini",              # = DEPLOYMENT name
    messages=[{"role": "user", "content": "Explain a Service Bus DLQ in one line"}],
    temperature=0.2,
)
print(resp.choices[0].message.content, resp.usage)
```

`DefaultAzureCredential` = your `az login` locally, managed identity in Azure — same code everywhere.

Useful CLI:

```bash
az cognitiveservices account list -o table
az cognitiveservices model list -l centralindia -o table          # what's available
az cognitiveservices usage list -l centralindia -o table          # quota used/limit
az cognitiveservices account deployment list -g RG -n NAME -o table
az cognitiveservices account list-deleted -o table
az cognitiveservices account purge -g RG -n NAME -l LOC
```

---

## 6. Gotchas

1. **Deployment name ≠ model name.** Code calls the deployment. Name deployments after the model (or a stable alias like `chat-default`) and keep it consistent across envs.
2. **Quota is per region per subscription** — dev using all TPM in a shared subscription starves another app. Separate subscriptions per env or carefully split capacity.
3. **Not every model is in every region / deployment type.** Check `model list` before choosing a region; India regions often get new models later than East US 2 / Sweden Central — Global Standard helps.
4. **Model retirement** — old versions are retired on a published schedule; auto-upgrade can change behaviour silently. Run evals before/after.
5. **Soft delete** — deleted Foundry/OpenAI resources keep the name ~48 days. Purge before recreating same name.
6. **Custom subdomain required** for Entra ID auth (`customSubDomainName`) and can't be changed later.
7. **Private endpoint DNS** — three privatelink zones may be involved. Missing one → some SDK calls (e.g. agents/projects endpoint) fail while chat works.
8. **Streaming through APIM/App Gateway/ingress** — response buffering breaks streaming; configure buffering off and long timeouts.
9. **Content filter false positives** (e.g. security/medical text flagged) → HTTP 400 with `content_filter`. Tune a custom content filter policy; don't just disable.
10. **Reasoning models** have different parameters (`max_completion_tokens`, no temperature) and bill hidden reasoning tokens.
11. **Serial deployments** — creating several model deployments on one account in parallel in IaC can conflict; chain with `dependsOn`.
12. **Role confusion** — `Cognitive Services Contributor` manages the resource but can't necessarily call models with Entra; `Cognitive Services OpenAI User` can call models; `Azure AI User` / `Azure AI Project Manager` are Foundry project roles. Check current role names in the docs for agents.
13. **Logging prompts** may log PII — mask before App Insights; set retention.

---

## 7. Production scenario FAQ

**Q1. Peak-hour 429s even though monthly usage is low.**
TPM is per minute. Spread load: multiple deployments/regions behind APIM with retry on 429 to next backend, increase TPM quota, move to Global Standard, or PTU with spillover for predictable peaks. Client side: exponential backoff honoring `retry-after-ms`.

**Q2. Client in a regulated sector requires data to stay in India.**
Use a Regional **Standard** or **Regional Provisioned** deployment in an India region that has the required model; avoid Global/Data Zone types. Document which models are available there; may need to pick an older or smaller model. Store data (index, files) in India regions too.

**Q3. Model version is being retired next month. Plan?**
Deploy the new version as a separate deployment → run evaluation suite on golden dataset → shadow/canary traffic via APIM → switch alias/deployment name in config → monitor quality/latency/cost → remove old deployment.

**Q4. Security finds API keys in app config.**
Rotate keys, set `disableLocalAuth: true`, move apps to managed identity + `Cognitive Services OpenAI User`, use Azure Policy to deny resources with local auth enabled.

**Q5. Agent works in dev, fails in prod with network-secured setup.**
Typical causes: missing subnet delegation for the agent subnet, private DNS zones not linked, project's managed identity missing roles on Storage/Cosmos/AI Search connections, or firewall blocking required outbound endpoints. Check activity log + agent run error details.

**Q6. How do you estimate cost for a chatbot?**
`requests/day × (avg input tokens × input price + avg output tokens × output price)`. Add embeddings for ingestion, AI Search tier, APIM, App Insights ingestion. Measure real token averages in staging; reasoning models and long histories change it drastically.

**Q7. Team wants an open model (Llama/Mistral) — how is it hosted?**
Either serverless (pay-per-token, Microsoft-hosted, like Azure OpenAI) where available, or managed compute (you pay for GPU VMs hourly, need GPU quota). Serverless first unless you need custom weights/fine-tunes or isolation.
