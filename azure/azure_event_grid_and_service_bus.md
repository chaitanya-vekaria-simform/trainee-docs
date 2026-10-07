# Azure Event Grid & Azure Service Bus — DevOps Notes

> Two messaging services that are often confused. One-liner:
> **Event Grid = "something happened" (notify, push, fan-out). Service Bus = "please do this" (reliable work queue, business messages).**
> IaC: Bicep lab Exercise 8 wires both together.

---

## 1. Events vs messages

```
 EVENT  (Event Grid)                         MESSAGE / COMMAND  (Service Bus)
 ─────────────────────                       ───────────────────────────────
 "Blob invoice-42.pdf was created"           "Process order 42, amount 70000"
 Lightweight notification, a FACT            Carries the business payload
 Publisher doesn't care who listens          Sender expects it to be handled exactly once-ish
 Fan-out to many handlers                    Competing consumers, ordering, transactions
 Push to handler (webhook/Function)          Consumer PULLS when ready (peek-lock)
```

### Azure messaging family

| Service | Pattern | Think of it as | Use for |
|---------|---------|----------------|---------|
| **Event Grid** | Pub/sub events, push (+pull in namespaces), MQTT | Doorbell / notification router | Reacting to Azure resource events, webhooks, lightweight integration |
| **Service Bus** | Queues & topics, enterprise broker | Post office with tracking, PO boxes, returns desk | Orders, payments, workflows, decoupling microservices |
| **Event Hubs** | Streaming log, partitions, replay | Kafka | Telemetry, logs, clickstream at millions/sec |
| **Storage Queues** | Simple queue | Basic mailbox | Cheap simple background jobs (> 80 GB backlog, no features) |

AWS mapping: Event Grid ≈ **EventBridge** (+ SNS for fan-out), Service Bus queues ≈ **SQS**, Service Bus topics ≈ **SNS + SQS**, Event Hubs ≈ **Kinesis / MSK**.

---

## 2. Event Grid

### Architecture

```
 SOURCES / PUBLISHERS                 EVENT GRID                         HANDLERS
 ┌──────────────────┐        ┌──────────────────────────┐        ┌────────────────────┐
 │ Storage account  │──┐     │  Topic                    │   ┌───►│ Azure Function     │
 │ Resource group   │  │     │   ├─ subscription A ──────┼───┘    ├────────────────────┤
 │ Key Vault        │  ├────►│   │   filter: BlobCreated │───────►│ Service Bus queue  │
 │ ACR, AKS, IoT... │  │     │   ├─ subscription B ──────┼───────►│ Webhook (HTTPS)    │
 │ Your app (custom)│──┘     │   │   filter: *.pdf       │        ├────────────────────┤
 └──────────────────┘        │   └─ dead-letter → blob   │───────►│ Event Hubs / Queue │
                             └──────────────────────────┘        │ Logic App          │
                                                                  └────────────────────┘
```

### Terms

| Term | Meaning |
|------|---------|
| **Event** | Small JSON (≤ 1 MB, billed per 64 KB) describing what happened: `id`, `source/topic`, `subject`, `type`, `time`, `data`. |
| **Event schema** | **CloudEvents 1.0** (open standard, recommended) or Event Grid schema. |
| **Publisher / source** | Who emits the event. |
| **System topic** | Topic for Azure service events (Storage, Key Vault, Resource Groups, ACR...). You create it on the source resource. |
| **Custom topic** | Your app publishes its own events to an endpoint. |
| **Domain** | Container for thousands of topics (multi-tenant SaaS: a topic per customer). |
| **Partner topic** | Events from SaaS partners (e.g. Auth0, SAP). |
| **Namespace** (newer) | Event Grid namespace: **pull delivery** (consumers fetch events like a queue), **MQTT broker** for IoT, namespace topics. |
| **Event subscription** | "Send events of type X matching filter Y to endpoint Z." |
| **Filters** | Event type, subject begins/ends with, advanced filters on `data` fields. |
| **Handler / endpoint** | Function, webhook, Service Bus queue/topic, Event Hubs, Storage queue, Logic App, Relay. |
| **Retry policy** | Exponential backoff up to **24h** (default) / max 30 attempts by default. |
| **Dead-lettering** | Undeliverable events written to a blob container (only if configured!). |
| **Webhook validation** | Handshake proving you own the endpoint before delivery starts. |
| **Delivery guarantee** | **At-least-once**, **no ordering guarantee** → handlers must be idempotent. |
| **Managed identity delivery** | Event Grid uses its identity to write to Service Bus/Event Hubs/Storage (needed when those block keys/public access). |

### Event example (CloudEvents)

```json
{
  "specversion": "1.0",
  "type": "Microsoft.Storage.BlobCreated",
  "source": "/subscriptions/.../storageAccounts/stdocs",
  "subject": "/blobServices/default/containers/uploads/blobs/invoice-42.pdf",
  "id": "9aeb0fdf-c01e-0131-0922-9eb54906e209",
  "time": "2026-10-07T10:15:00Z",
  "data": { "api": "PutBlob", "contentType": "application/pdf", "contentLength": 52411,
            "url": "https://stdocs.blob.core.windows.net/uploads/invoice-42.pdf" }
}
```

### DevOps use cases

- Blob uploaded → trigger indexing / virus scan / thumbnail (RAG ingestion: new doc → re-index).
- **ACR image pushed** → trigger deployment pipeline / scan.
- **Key Vault secret near expiry** → Function rotates it / alert.
- **Resource group events** (resource created/deleted) → governance, tagging enforcement, audit to Teams/Slack.
- **AKS events** (new Kubernetes version available) → upgrade notifications.
- Policy state changes → compliance notifications.

---

## 3. Service Bus

### Architecture

```
                      Service Bus NAMESPACE  (sb-prod.servicebus.windows.net)
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │  QUEUE "orders"  (point-to-point, competing consumers)                       │
 │   producer ──► [m5][m4][m3][m2][m1] ──► consumer 1                           │
 │                                     └─► consumer 2   (each msg to ONE)       │
 │                      └─► $DeadLetterQueue  (poison / expired messages)       │
 │                                                                              │
 │  TOPIC "order-events"  (pub/sub)                                             │
 │   producer ──► topic ──┬─► subscription "email"   (rule: all)  ──► email svc │
 │                        ├─► subscription "fraud"   (rule: amount > 50000)     │
 │                        └─► subscription "audit"   (rule: all)  ──► audit svc │
 │                each subscription = its own queue with its own DLQ            │
 └──────────────────────────────────────────────────────────────────────────────┘
```

### Terms

| Term | Meaning |
|------|---------|
| **Namespace** | Container/endpoint for queues & topics; tier, networking, auth set here. |
| **Queue** | FIFO-ish store; one consumer gets each message. |
| **Topic** | Publish once, copy to many **subscriptions**. (Standard/Premium only.) |
| **Subscription** | Virtual queue on a topic with **rules** (SQL filter, correlation filter) and optional actions. |
| **Message** | Body (up to 256 KB Standard, up to 100 MB Premium) + system properties + **application properties** (used by filters). |
| **Peek-lock** (default) | Receive → message locked (hidden) → **Complete** (delete) / **Abandon** (release) / **Dead-letter** / **Defer**. Safe. |
| **Receive-and-delete** | Deleted on receive — faster, message lost if consumer crashes. |
| **Lock duration** | How long the consumer holds the lock (default 1 min, max 5 min). Long jobs must **renew** the lock. |
| **Delivery count / MaxDeliveryCount** | Each abandon/lock expiry +1; at max (default **10**) → DLQ. |
| **Dead-letter queue (DLQ)** | Sub-queue `<queue>/$DeadLetterQueue` for poison, expired, filter-error messages. **Never auto-processed** — you must monitor and handle it. |
| **TTL (time to live)** | Message expiry; optional dead-letter on expiry. |
| **Sessions** | Group related messages (`SessionId`, e.g. orderId) → **ordered** processing by one consumer at a time. |
| **Duplicate detection** | Drop messages with same `MessageId` within a time window. |
| **Scheduled messages** | Deliver at a future time. |
| **Deferral** | Set aside a message to fetch later by sequence number. |
| **Transactions** | Atomic receive + send across entities in the same namespace. |
| **Auto-forwarding** | Chain queue/subscription → another queue/topic. |
| **Prefetch** | Client pulls batches ahead — faster, but locks may expire while waiting. |
| **SAS policy** | Key-based auth (Manage/Send/Listen) — prefer Entra ID roles: *Azure Service Bus Data Sender / Receiver / Owner*. |
| **Messaging units (MU)** | Premium capacity units (1, 2, 4, 8, 16); dedicated resources. |
| **Geo-disaster recovery** | Premium: alias pairing of namespaces (metadata only). **Geo-replication** (Premium, newer): replicates metadata + data. |

### Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Queues | ✅ | ✅ | ✅ |
| Topics/subscriptions | ❌ | ✅ | ✅ |
| Sessions, transactions, duplicate detection | ❌ | ✅ | ✅ |
| Max message size | 256 KB | 256 KB | up to 100 MB |
| Isolation / predictable perf | Shared | Shared (throttling possible) | Dedicated MUs |
| VNet / private endpoint | ❌ | ❌ | ✅ |
| Availability zones, geo-DR / geo-replication | — | Limited | ✅ |
| Pricing | Per operation | Base + per operation | Per MU per hour |

> **Private endpoints require Premium** — a common surprise in security reviews.

---

## 4. Event Grid vs Service Bus — decision

```
 Is it a notification that something already happened, from an Azure service?
     └─ yes ─► Event Grid (system topic)
 Do you need high-value business messages, ordering, transactions, retries under consumer control?
     └─ yes ─► Service Bus
 Millions of telemetry events/sec, replay, stream processing?
     └─ yes ─► Event Hubs
 Many subscribers, push to webhooks/Functions, filtering, low cost per event?
     └─ yes ─► Event Grid (custom topic)
 Both? (very common)
     └─ Event Grid ──► Service Bus queue ──► worker  (buffer pushes into a durable queue)
```

| | Event Grid | Service Bus |
|---|---|---|
| Model | Push (+pull in namespaces) | Pull (consumer controls rate) |
| Payload | Small event (≤1 MB) | Full business message (256 KB–100 MB) |
| Ordering | No | Yes (sessions) |
| Delivery | At-least-once | At-least-once (peek-lock), duplicate detection |
| Retry | Event Grid retries for up to 24h | Consumer abandons → redelivery up to MaxDeliveryCount |
| Poison handling | Dead-letter to blob (if configured) | Built-in DLQ |
| Back-pressure | Handler must keep up (or throttles/retries) | Queue absorbs spikes |
| Cost | Per operation, very cheap | Tier-based |

---

## 5. Gotchas

**Event Grid**

1. **No dead-letter by default** — undeliverable events are dropped after retries. Always configure a dead-letter container in prod.
2. **At-least-once + no ordering** — handlers must be idempotent (dedupe on event `id`).
3. **Webhook validation** fails if your endpoint (Function/APIM/App Gateway) doesn't answer the handshake; Azure Functions Event Grid trigger handles it automatically.
4. **System topic per source** — one per storage account; creating via portal auto-creates names that IaC later conflicts with. Define system topics explicitly in IaC.
5. **Private destinations** — delivering to a private-endpoint Service Bus/Event Hubs requires **managed identity delivery** + "trusted Microsoft services" allowed.
6. **Storage events fire for every write**, including SDK multi-part uploads and overwrites → filter by `api`/type, subject suffix.
7. Event Grid handler too slow → 4xx/5xx/timeout → retries → duplicates. Put a Service Bus queue in between.

**Service Bus**

8. **DLQ silently fills up** — nothing alerts by default. Alert on `DeadletteredMessages` metric > 0.
9. **Lock lost** errors (`MessageLockLost`) → processing time > lock duration. Renew locks (SDK auto-renew), lower prefetch, or reduce batch.
10. **Throttling on Standard** (`ServerBusy`) under noisy-neighbour load → Premium for prod workloads with SLAs.
11. **Topic subscription with no rule matches** → message silently not delivered there. Default rule is `1=1` (all); deleting it drops everything unless new rules added.
12. **Sessions on a queue** → consumers without session receivers can't read it; every message needs `SessionId`.
13. **Changing `requiresSession`, `requiresDuplicateDetection`, partitioning** requires recreating the entity — data loss if not drained.
14. **Basic tier** has no topics — fails IaC deployment.
15. **SAS connection strings in app settings** → rotate & leak risk. Use managed identity + `disableLocalAuth: true`.
16. **Max delivery count too high** → poison message blocks a session / burns compute; too low → transient failures land in DLQ.

---

## 6. Production scenario FAQ

**Q1. Orders are being processed twice.**
At-least-once delivery: consumer crashed or lock expired after doing the work but before `Complete`. Fix: idempotent consumer (dedupe table on `MessageId`/orderId), enable duplicate detection for producer retries, renew locks for long processing, complete only after commit.

**Q2. DLQ has 30,000 messages after a deployment.**
New consumer version threw exceptions → each message hit MaxDeliveryCount. Steps: check `DeadLetterReason`/`DeadLetterErrorDescription`, roll back or fix consumer, then **resubmit** DLQ messages (Service Bus Explorer, a small replay tool/Function) — after fixing. Add alert on DLQ count and a runbook.

**Q3. Messages for the same customer are processed out of order.**
Use **sessions** with `SessionId = customerId`; consumers use session receivers. Ordering is per session, parallelism across sessions.

**Q4. Security review: Service Bus must not be public.**
Requires **Premium** tier → private endpoint + `publicNetworkAccess: Disabled` + private DNS `privatelink.servicebus.windows.net`. Event Grid delivering to it must use managed identity + trusted services. Budget for Premium MU cost.

**Q5. Event Grid → Function webhook loses events during Function outage.**
Retries for up to 24h help, but configure **dead-letter** to blob and alert. Better: Event Grid → Service Bus queue → Function (queue trigger), so the queue buffers during outages and you control retry.

**Q6. Region outage — what about messages?**
Service Bus Premium **geo-replication** replicates data; older **Geo-DR** replicates only metadata (in-flight messages in primary are stuck until it returns). Clients use the alias FQDN. Test failover. Event Grid: custom topics have built-in regional failover options (geo-DR to paired region) — check per-resource settings; system topics follow their source resource.

**Q7. How do you monitor these?**
Service Bus metrics: `ActiveMessages`, `DeadletteredMessages`, `ServerErrors`, `ThrottledRequests`, `IncomingMessages` vs `CompletedMessages` (backlog growing?). Event Grid: `PublishFailCount`, `DeliveryFailCount`, `DroppedEventCount`, `DeadLetteredCount`, `MatchedEventCount`. Diagnostic logs to Log Analytics; alerts with action groups.

**Q8. KEDA on AKS scaling consumers?**
KEDA `azure-servicebus` scaler scales pods on queue/subscription length (with workload identity). Set `messageCount` target per pod, min/max replicas, and make pods handle SIGTERM gracefully (finish/abandon in-flight messages).

---

## 7. CLI quick reference

```bash
# Service Bus
az servicebus namespace create -g RG -n sb-lab --sku Standard
az servicebus queue create -g RG --namespace-name sb-lab -n orders --max-delivery-count 10
az servicebus topic create -g RG --namespace-name sb-lab -n order-events
az servicebus topic subscription create -g RG --namespace-name sb-lab --topic-name order-events -n email
az servicebus topic subscription rule create -g RG --namespace-name sb-lab --topic-name order-events \
  --subscription-name fraud -n high --filter-sql-expression "amount > 50000"
az servicebus queue show -g RG --namespace-name sb-lab -n orders --query countDetails

# Event Grid
az eventgrid system-topic create -g RG -n egst-stg --location centralindia \
  --topic-type Microsoft.Storage.StorageAccounts --source <storage-id>
az eventgrid system-topic event-subscription create -g RG --system-topic-name egst-stg -n to-sb \
  --endpoint-type servicebusqueue --endpoint <queue-id> \
  --included-event-types Microsoft.Storage.BlobCreated \
  --deadletter-endpoint <storage-id>/blobServices/default/containers/deadletter
az eventgrid topic create -g RG -n egt-app -l centralindia
```
