# AI Terms for a DevOps Engineer — Beginner Guide

> Goal: understand enough AI vocabulary to **deploy, secure, scale, monitor and debug** AI apps on Azure — without needing to be a data scientist.
> Read in order: Sections 1–4 are the foundation; the rest build on them.
> Next: [`02_azure_openai_and_microsoft_foundry.md`](02_azure_openai_and_microsoft_foundry.md) → [`03_azure_ai_search.md`](03_azure_ai_search.md)

---

## 1. The big picture

```
 Artificial Intelligence (AI)  — machines doing "smart" tasks
   └── Machine Learning (ML)   — learn patterns from data instead of hand-written rules
         └── Deep Learning     — ML using large neural networks
               └── Generative AI (GenAI) — models that CREATE text, images, code, audio
                     └── Large Language Models (LLMs) — GenAI for text (GPT, Claude, Llama, Phi...)
```

Old way vs ML:

```
 Traditional code:   rules + data  ──►  answer        (if amount > 50000 then fraud)
 Machine learning:   data + answers ──► rules (model) (learn what fraud looks like)
```

As a DevOps engineer you mostly deal with **using** pre-trained models via APIs, not training them. Your job looks like any other API-heavy app: networking, identity, quotas, cost, latency, monitoring — plus a few AI-specific ideas below.

---

## 2. Model basics

| Term | Plain-English meaning | DevOps angle |
|------|----------------------|--------------|
| **Model** | A file of learned numbers (weights) that turns input into output. | You deploy/consume it, like a container image. |
| **Parameters / weights** | The learned numbers. "70B model" = 70 billion parameters. | Bigger = smarter but slower, costlier, needs more GPU memory. |
| **Training** | Teaching a model from data. Very expensive (thousands of GPUs, weeks). | Rarely your job. |
| **Inference** | Using a trained model to get an answer. | This is what you run & pay for. "Inference endpoint" = API you call. |
| **Pre-trained / foundation model** | A general model trained by OpenAI/Microsoft/Meta etc. | You pick one from a catalog. |
| **Fine-tuning** | Extra training on your data to specialize a model. | Costs training + hosting; usually try prompts/RAG first. |
| **Open-weight model** | Weights downloadable (Llama, Mistral, Phi). | Can self-host on AKS/VMs with GPUs. |
| **Closed model** | Only via API (GPT-4.x/5, Claude). | No GPUs to manage; quota & rate limits instead. |
| **SLM (Small Language Model)** | Smaller, cheaper, faster (e.g. Phi family). | Can run on CPU/edge; good for simple tasks. |
| **Multimodal** | Understands more than text — images, audio, video. | Larger payloads, different pricing. |
| **Reasoning model** | Model that "thinks" internally before answering (o-series, GPT-5 reasoning). | Higher latency and more (hidden) tokens billed. |
| **GPU** | Hardware that runs models fast. | AKS GPU node pools, quota requests, very expensive per hour. |

---

## 3. Tokens — the unit of everything

A **token** ≈ a piece of a word. Roughly **1 token ≈ 4 English characters ≈ ¾ of a word**.

```
 "DevOps engineers love automation"
   → ["Dev", "Ops", " engineers", " love", " automation"]   = 5 tokens
```

| Term | Meaning |
|------|---------|
| **Input / prompt tokens** | What you send (instructions + question + retrieved documents + chat history). |
| **Output / completion tokens** | What the model generates. Usually priced higher than input. |
| **Context window** | Max tokens (input + output) per request, e.g. 128K, 1M. Like RAM for a conversation. |
| **Max tokens / max_output_tokens** | Limit you set on the answer length. |
| **TPM** | Tokens Per Minute — the main **quota** unit on Azure OpenAI. |
| **RPM** | Requests Per Minute — the second quota limit. |
| **Cached tokens** | Repeated prompt prefixes billed cheaper (prompt caching). |

Why DevOps cares:

- **Cost** = tokens × price. Long chat history and huge retrieved documents = big bills.
- **Throttling** = exceeding TPM/RPM → HTTP **429 Too Many Requests**. Needs retry with backoff, load balancing, or more quota.
- **Latency** grows with output tokens (models generate one token at a time).

---

## 4. Prompts & conversation

| Term | Meaning |
|------|---------|
| **Prompt** | The input text you send. |
| **System prompt / system message / instructions** | Hidden instructions that set the model's behaviour ("You are a support bot for X. Answer only from provided docs."). |
| **User message** | What the end user typed. |
| **Assistant message** | What the model replied (kept in history for multi-turn chat). |
| **Prompt engineering** | Writing prompts that give reliable results (clear instructions, examples, format). |
| **Zero-shot / few-shot** | No examples vs a few examples in the prompt. |
| **Temperature** | Randomness 0–2. Low (0–0.3) = consistent, factual. High = creative. Some reasoning models fix it. |
| **Top-p** | Another randomness knob; change temperature *or* top-p, not both. |
| **Streaming** | Receive the answer token-by-token (SSE). Needs proxies/gateways that don't buffer responses. |
| **Structured output / JSON mode** | Force the model to answer in a JSON schema — easier to parse in code. |
| **Hallucination** | Confident but wrong/made-up answer. Main reason for RAG and grounding. |
| **Grounding** | Making the model answer from given facts (your docs) instead of its memory. |
| **Stateless** | The model remembers nothing between API calls; the app re-sends history each time. |

A chat request on the wire:

```json
{
  "model": "gpt-4.1-mini",
  "temperature": 0.2,
  "max_tokens": 500,
  "messages": [
    { "role": "system",    "content": "You answer DevOps questions in 3 bullet points." },
    { "role": "user",      "content": "What is a readiness probe?" }
  ]
}
```

---

## 5. Embeddings & vectors (needed for search/RAG)

**Embedding** = turning text into a list of numbers (a **vector**) that represents its *meaning*.

```
 "reset my password"       → [0.12, -0.80, 0.33, ... 1536 numbers]
 "forgot login credentials"→ [0.10, -0.78, 0.35, ...]   ← very close = similar meaning
 "kubernetes pod crash"    → [-0.60, 0.21, -0.05, ...]  ← far away
```

| Term | Meaning |
|------|---------|
| **Embedding model** | Model that produces vectors (e.g. `text-embedding-3-small`, 1536 dimensions). |
| **Dimensions** | Length of the vector. More = more detail, more storage. |
| **Vector database / vector store / vector index** | Stores vectors and finds nearest ones fast (Azure AI Search, Cosmos DB, PostgreSQL pgvector, Redis). |
| **Similarity / cosine similarity** | How "close" two vectors are. |
| **Nearest neighbour (kNN / ANN, HNSW)** | Algorithm to find the most similar vectors. ANN/HNSW = approximate but fast. |
| **Keyword search (BM25 / full-text)** | Classic search matching words. Great for codes, IDs, exact names. |
| **Vector / semantic search** | Search by meaning. Great for natural questions. |
| **Hybrid search** | Keyword + vector combined → usually best results. |
| **Reranker / semantic ranker** | Second model that re-orders the top results for relevance. |
| **Chunking** | Splitting big documents into smaller pieces (e.g. 500–1000 tokens with overlap) before embedding. |

Gotcha: **query and documents must use the same embedding model**. Changing the model = re-embed (re-index) everything.

---

## 6. RAG — Retrieval-Augmented Generation (most common enterprise pattern)

Problem: the model doesn't know your company's docs and may hallucinate.
Solution: **search your docs first, then give the relevant pieces to the model with the question.**

```
          INGESTION (offline, batch / event-driven)
 ┌───────────┐   ┌─────────┐   ┌───────────┐   ┌────────────────┐
 │ PDFs/docs │──►│ chunk   │──►│ embed     │──►│ Azure AI Search │
 │ blob/SP   │   │ (split) │   │ (vectors) │   │ index           │
 └───────────┘   └─────────┘   └───────────┘   └───────▲────────┘
                                                       │
          QUERY (online, per user question)            │ 2. hybrid search (top 5 chunks)
 ┌──────┐ 1. question ┌──────────┐ ────────────────────┘
 │ User │────────────►│  App /   │
 │      │◄────────────│  API     │── 3. prompt = instructions + chunks + question ──►┌─────────┐
 └──────┘ 5. answer   └──────────┘◄─ 4. grounded answer + citations ────────────────│  LLM    │
                                                                                     └─────────┘
```

RAG vs fine-tuning:

| Need | Use |
|------|-----|
| Answer from company docs that change often | **RAG** |
| Change style/format/tone, narrow task | Fine-tuning (after prompts fail) |
| Both | RAG + fine-tuned model |

DevOps responsibilities in RAG: storage + indexing pipeline, Search service sizing, private networking, managed identities between services, index rebuilds, keeping docs' **permissions** (security trimming) so users only retrieve what they're allowed to see.

---

## 7. Agents & tools

| Term | Meaning |
|------|---------|
| **Function calling / tool calling** | The model replies "call `get_order(id=42)`" instead of text; your code runs it and sends back the result. |
| **Agent** | An LLM in a loop: think → choose a tool → observe result → repeat until done. |
| **Tools** | Things an agent can use: search, code interpreter, APIs, databases, other agents. |
| **MCP (Model Context Protocol)** | Open standard for exposing tools/data to AI apps/agents (like a USB-C port for tools). |
| **A2A (Agent-to-Agent)** | Protocol for agents talking to other agents. |
| **Multi-agent** | Several specialized agents coordinating (planner, researcher, coder). |
| **Orchestration framework** | Code libraries for building agents: Microsoft Agent Framework (successor to Semantic Kernel + AutoGen), LangChain/LangGraph, LlamaIndex. |
| **Memory / thread** | Stored conversation state for an agent (often in Cosmos DB). |
| **Human-in-the-loop** | Agent pauses for approval before risky actions. |

```
 User: "Restart the failing pod in prod and tell me why it failed"
   │
   ▼
 Agent loop ──► tool: kubectl get pods        ──► sees CrashLoopBackOff
   │       ──► tool: kubectl logs            ──► OOMKilled
   │       ──► needs approval to restart     ──► human approves
   │       ──► tool: kubectl rollout restart
   ▼
 "Restarted. Root cause: memory limit 256Mi too low."
```

DevOps angle: agents **take actions** → least-privilege identities, audit logs, approval gates, rate limits, timeouts, cost caps.

---

## 8. Safety, evaluation & operations (LLMOps / GenAIOps)

| Term | Meaning |
|------|---------|
| **Responsible AI** | Microsoft's principles: fairness, reliability/safety, privacy/security, inclusiveness, transparency, accountability. |
| **Content filter / content safety** | Classifies prompts & answers for hate, violence, sexual, self-harm content; blocks above a threshold. On by default in Azure OpenAI. |
| **Prompt injection / jailbreak** | User (or a document!) tries to override system instructions ("ignore previous instructions..."). **Prompt Shields** detect it. |
| **Indirect prompt injection** | Malicious instructions hidden inside retrieved docs/web pages. Big risk for RAG and agents. |
| **Groundedness** | Is the answer supported by the provided sources? Can be measured. |
| **Evaluation (evals)** | Automated tests for AI output quality: relevance, groundedness, coherence, safety. The "unit tests" of GenAI. |
| **Red teaming** | Attacking your own AI app to find unsafe behaviour. |
| **Tracing / observability** | Recording each step (prompt, tool call, tokens, latency) — OpenTelemetry → Application Insights. |
| **LLMOps / GenAIOps** | DevOps for AI apps: version prompts, run evals in CI, monitor tokens/latency/quality, roll out models safely. |
| **AI gateway** | API Management in front of models: auth, token rate limiting, load balancing across regions, usage logging per team. |
| **Model retirement** | Model versions get retired on a schedule → planned upgrades are a recurring ops task. |
| **Data residency** | Where prompts are processed (Global vs Data Zone vs Regional deployments). |
| **PII** | Personal data — mask/redact before sending to models/logs. |

---

## 9. Classic ML terms you'll hear

| Term | Meaning |
|------|---------|
| Dataset / training / validation / test set | Data used to train and check a model |
| Feature | An input column (age, amount) |
| Label | The correct answer in training data |
| Classification / regression | Predict a category / a number |
| Overfitting | Memorizes training data, fails on new data |
| Accuracy, precision, recall, F1 | Quality metrics |
| MLOps | DevOps for ML models (Azure Machine Learning: pipelines, registries, endpoints) |
| Model registry | Versioned store of models (like a container registry) |
| Managed online endpoint | Hosted model API on Azure ML |
| Batch inference | Run a model over lots of data offline |
| Drift (data/model) | Real-world data changes → model quality drops |
| ONNX | Portable model format |
| Hugging Face | Huge public hub of open models/datasets |

---

## 10. Azure AI service map (where each term lives)

```
 ┌────────────────────────────── Microsoft Foundry ─────────────────────────────────┐
 │  Model catalog (OpenAI GPT, Phi, Llama, Mistral, DeepSeek, Grok...)               │
 │  Azure OpenAI deployments  │ Agent Service │ Evaluations │ Tracing │ Content Safety │
 │  Azure AI services (newer docs: "Foundry Tools"): Speech, Vision, Language,       │
 │      Translator, Document Intelligence, Content Understanding                     │
 └───────────────────────────────────────────────────────────────────────────────────┘
        │ uses                         │ uses                          │ uses
        ▼                              ▼                               ▼
  Azure AI Search                 Storage / Cosmos DB             Key Vault, App Insights,
  (vector + hybrid + RAG)         (docs, chat history)            APIM (AI gateway), VNet/PE

  Azure Machine Learning  → classic ML / training / MLOps (custom models)
  Copilot Studio          → low-code agents for business users
```

| Need | Azure service |
|------|---------------|
| Chat/completions/embeddings with GPT models | Azure OpenAI (inside Foundry) |
| Build/host agents | Foundry Agent Service |
| Search your docs (RAG) | Azure AI Search |
| Extract data from invoices/forms | Document Intelligence |
| Speech-to-text, text-to-speech | Azure AI Speech |
| Detect harmful content / prompt attacks | Azure AI Content Safety |
| Train custom ML models | Azure Machine Learning |
| Rate-limit/load-balance AI APIs | API Management (AI gateway policies) |
| Event-driven ingestion pipeline | Event Grid + Functions / Service Bus |

---

## 11. Beginner FAQ (production flavoured)

**Q1. Users get "429 Too Many Requests" from the chatbot at 10 AM daily.**
You're hitting TPM/RPM quota. Short term: retry with exponential backoff honoring `retry-after`. Medium: increase quota or use a Global Standard deployment; put APIM in front to load-balance across multiple deployments/regions. Long term: provisioned throughput (PTU) for predictable heavy load.

**Q2. The bot invents policy details that don't exist.**
Hallucination. Use RAG with good retrieval (hybrid + semantic ranker), instruct "answer only from sources, say I don't know otherwise", lower temperature, and add groundedness evaluations to CI.

**Q3. Monthly AI bill doubled without more users.**
Check token usage per call: growing chat history, larger top-K chunks, a switch to a reasoning model (hidden reasoning tokens), or a retry storm. Fix: trim history/summarize, smaller model for simple tasks, prompt caching, per-team usage tracking via APIM.

**Q4. Security asks: "Is our data used to train OpenAI's models?"**
Azure OpenAI: prompts/completions are not used to train the base models and aren't shared with OpenAI. Still: private endpoints, disable keys (Entra ID only), customer-managed keys if required, and check abuse-monitoring data retention options for your compliance needs.

**Q5. We changed the embedding model and search results became garbage.**
Query vectors (new model) and stored vectors (old model) are incompatible. Re-embed and rebuild the index; do it blue/green (new index, switch alias/app config, delete old).

**Q6. What should I learn first as a DevOps engineer?**
(1) Tokens, quotas, 429s. (2) Deploying a model in Foundry and calling it with Entra ID. (3) RAG with Azure AI Search. (4) Private networking + managed identities for these services. (5) APIM AI gateway. (6) Tracing/evals in CI. Bicep for all of it — see the lab Exercise 9.
