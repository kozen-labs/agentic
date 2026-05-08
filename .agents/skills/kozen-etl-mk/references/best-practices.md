---
name: "@kozen/etl-mk — Best Practices"
description: >
  Production best practices for @kozen/etl-mk: delegate design (idempotency, error handling,
  structured logging), at-least-once delivery implications, Kafka message key strategy for
  ordering guarantees, consumer group ID uniqueness, SSL in production, DLQ monitoring and
  reprocessing, writeMode selection, silent failure patterns to avoid, secrets in delegates
  via @kozen/secret, and checklist for production readiness.
category: kozen-etl-mk
tags:
  - kozen
  - etl
  - best-practices
  - idempotency
  - at-least-once
  - DLQ
  - delegate-design
  - kafka-key
  - consumer-group
  - SSL
  - production
  - structured-logging
---

# @kozen/etl-mk — Best Practices

## 1. Delegate design

### Keep delegates pure transforms

A delegate should receive data in, return data out. Avoid side effects beyond logging.
Side effects (calling external APIs, writing to other collections) belong outside the return
path so they don't block the pipeline if they fail.

```javascript
// Good — pure transform
export async function insert(change, tools) {
  const doc = change.fullDocument;
  return { id: doc._id.toString(), status: doc.status, ts: new Date().toISOString() };
}

// Risky — side effect inside the return path blocks offset commit if it fails
export async function insert(change, tools) {
  await callExternalApi(change.fullDocument); // if this throws, offset is not committed
  return { id: change.fullDocument._id.toString() };
}
```

If you need a side effect, wrap it in try/catch and log the error — do not let it propagate
to the pipeline unless you explicitly want to trigger the retry/DLQ path.

### Use `return null` to filter events

Returning `null` or `undefined` from any handler skips the event entirely — no publish,
no write, no DLQ. This is the correct way to filter events you don't want to forward.

```javascript
export async function update(change, tools) {
  // Only forward status changes — ignore all other field updates
  if (!change.updateDescription?.updatedFields?.status) return null;
  return { id: change.documentKey._id.toString(), status: change.updateDescription.updatedFields.status };
}
```

### Handle errors explicitly in delegates

If a delegate throws, the error is caught by the pipeline and routed to the DLQ (after retries
on KM). Do not swallow errors silently — let the pipeline route them to the DLQ where they
can be inspected and reprocessed. Only catch errors when you have a meaningful recovery path.

```javascript
export async function message(msg, tools) {
  try {
    const enriched = await enrichFromCache(msg.id);
    return { ...msg, ...enriched, savedAt: new Date() };
  } catch (err) {
    // Recovery: store without enrichment rather than routing to DLQ
    tools.logger?.warn({ id: msg.id, err: err.message }, 'enrichment failed, saving raw');
    return { ...msg, savedAt: new Date(), enriched: false };
  }
}
```

### Use structured logging, not console.log

The Kozen logger emits structured JSON. `console.log` bypasses it and can corrupt MCP output
or make logs unsearchable in log aggregators.

```javascript
// Bad
console.log('processing message', msg.id);

// Good
tools.logger?.info({ id: msg.id, flow: tools.flow }, 'processing message');
tools.logger?.warn({ id: msg.id, reason: 'missing field' }, 'skipping event');
tools.logger?.error({ id: msg.id, err: err.message }, 'delegate error');
```

---

## 2. At-least-once delivery — implications for KM

The KM pipeline commits Kafka offsets **only after** a successful MongoDB write or DLQ routing.
This guarantees no message is permanently lost, but it also means **the same message can be
delivered more than once** after a crash and restart.

**What this means for your delegate:**
- Your delegate may receive the same Kafka message twice.
- The resulting MongoDB document will be written twice.
- If you use `writeMode=insert`, you get duplicate documents.

**How to handle it:**
1. **Use `writeMode=upsert` and include `_id` in your returned document** — MongoDB will
   update the existing document instead of inserting a duplicate.

   ```javascript
   export async function message(msg, tools) {
     return {
       _id:       msg.id,           // must be present for upsert to match
       ...msg,
       updatedAt: new Date()
     };
   }
   ```

   ```bash
   KOZEN_ETL_KM_DESTINATION_WRITE_MODE=upsert
   ```

2. **Use `writeMode=insert` only when duplicates are acceptable** — for append-only logs,
   audit trails, or time-series where a duplicate entry is harmless.

3. **Add a deduplication field** if you cannot use upsert — include a `messageId` or `eventId`
   field and create a unique index on it in MongoDB to reject duplicates at the database level.

---

## 3. Kafka message key strategy (MK)

The Kafka message key controls which partition a message lands in. **Messages with the same
key always go to the same partition**, preserving their relative order.

**Always set the key explicitly** via `tools.setMessageKey()`. Without a key, KafkaJS uses
round-robin partitioning — messages from the same document can arrive out of order at consumers.

```javascript
export async function insert(change, tools) {
  // Use the document _id as key — all events for the same document go to the same partition
  tools.setMessageKey(change.fullDocument._id.toString());
  return { ... };
}

export async function update(change, tools) {
  // documentKey._id is available even when fullDocument is not (for partial updates)
  tools.setMessageKey(change.documentKey._id.toString());
  return { ... };
}
```

**Key selection guide:**

| Goal | Key to use |
|---|---|
| Order events per document | Document `_id` |
| Order events per user | User ID field |
| Fan out evenly across partitions (no ordering needed) | Random UUID |
| Group related documents together | Tenant ID or shard key |

**Never use a high-cardinality random key** — it fills partition metadata and provides no
ordering benefit. If you don't need ordering, omit the key and let the default behavior apply.

---

## 4. Consumer group ID strategy (KM)

The consumer group ID determines which partition offsets are tracked independently.

**Rules:**
1. **One unique group ID per logical consumer pipeline.** If two pipelines share the same
   group ID and subscribe to the same topic, Kafka splits the partitions between them —
   each pipeline processes only a subset of messages.

2. **Change the group ID when you want to reprocess from the beginning.** Kafka tracks the
   last committed offset per group. A new group ID starts from the latest offset by default.
   To reprocess historical messages, use a new group ID or reset the offset via Kafka CLI.

3. **Use a descriptive, environment-scoped name.** Avoid generic names like `etl-group` —
   use `<project>-<env>-<purpose>`, e.g., `payments-prod-archive`.

```bash
KOZEN_ETL_KM_SOURCE_GROUP_ID=payments-prod-archive
```

---

## 5. DLQ configuration and monitoring

### Always configure DLQ topics in production

Without a DLQ, messages that exceed retry attempts are **permanently dropped**. The pipeline
logs the error and moves on. In production, this causes silent data loss.

```bash
# MK: failed publishes go to Kafka DLQ topic
KOZEN_ETL_MK_DLQ_TOPIC=orders.events-dlq

# KM: failed writes go to MongoDB DLQ collection
KOZEN_ETL_KM_DLQ_TOPIC=orders.events-dlq   # also used as MongoDB collection name
```

### DLQ envelope format

**MK DLQ** (Kafka topic `<topic>-dlq`):
```json
{
  "originalPayload": "{ \"_id\": \"...\", \"operationType\": \"insert\", ... }",
  "error": "TypeError: cannot read property 'status' of undefined",
  "flow": "abc-123",
  "ts": "2026-05-06T10:00:00.000Z"
}
```

**KM DLQ** (MongoDB collection `<destinationDb>.<topic>-dlq`):
```json
{
  "originalMessage": "{ \"id\": \"...\", \"amount\": 99 }",
  "error": "MongoServerError: E11000 duplicate key error",
  "flow": "abc-123",
  "ts": "2026-05-06T10:00:00.000Z"
}
```

### Reprocessing DLQ messages

DLQ messages are stored with the original payload intact. To reprocess:

1. **For Kafka DLQ topics** — consume the DLQ topic with a separate consumer, fix the
   payload if needed, and republish to the original topic.
2. **For MongoDB DLQ collections** — query the collection, fix the documents, and re-insert
   them into the source collection or trigger the pipeline manually.

Monitor DLQ depth as a key operational metric — a growing DLQ indicates systematic delegate
or infrastructure failures.

---

## 6. SSL in production

Always enable SSL when connecting to managed Kafka services (Confluent Cloud, MSK, etc.)
or when traffic crosses network boundaries.

```bash
# MK: MongoDB → Kafka
KOZEN_ETL_MK_DESTINATION_SSL=true

# KM: Kafka → MongoDB
KOZEN_ETL_KM_SOURCE_SSL=true
```

For mTLS or custom certificate authorities, configure KafkaJS SSL options programmatically
via the programmatic embedding pattern — environment variables only toggle the SSL boolean.

---

## 7. Retry configuration (KM)

The KM pipeline retries failed writes with linear backoff: `retryDelayMs × attemptNumber`.

| Setting | Default | Guidance |
|---|---|---|
| `KOZEN_ETL_KM_RETRY_ATTEMPTS` | `3` | Increase for transient network or Atlas M10 cold-start errors |
| `KOZEN_ETL_KM_RETRY_DELAY_MS` | `1000` | Increase for slow MongoDB writes under load |

Effective wait times with defaults (3 attempts, 1000 ms):
- Attempt 1: immediate
- Attempt 2: 1000 ms
- Attempt 3: 2000 ms
- After 3 failures → DLQ

```bash
# More resilient for unstable connections
KOZEN_ETL_KM_RETRY_ATTEMPTS=5
KOZEN_ETL_KM_RETRY_DELAY_MS=500
```

---

## 8. Secrets in delegates

Never hardcode credentials in delegate files. Use `@kozen/secret` to resolve them at runtime
from the Kozen IoC container.

```javascript
// delegates/enriched-archive.mjs
// Load both modules: KOZEN_MODULE_LOAD=@kozen/secret,@kozen/etl-mk

export async function message(msg, tools) {
  const secretMgr = await tools.assistant?.resolve('secret:manager');
  const apiKey = await secretMgr?.resolve('ENRICHMENT_API_KEY');

  const enriched = await fetch(`https://api.example.com/enrich?id=${msg.id}`, {
    headers: { Authorization: `Bearer ${apiKey}` }
  }).then(r => r.json());

  return { ...msg, ...enriched, savedAt: new Date() };
}
```

Start command:
```bash
KOZEN_MODULE_LOAD=@kozen/secret,@kozen/etl-mk \
KOZEN_SM_DRIVER=mdb \
MDB_URI=mongodb+srv://... \
MDB_MASTER_KEY=<base64-key> \
npx kozen --action=etl-mk:start --envFile=.env
```

---

## 9. Silent failure patterns to avoid

These mistakes cause the pipeline to start but process no events — with no obvious error:

| Mistake | Symptom | Fix |
|---|---|---|
| `DELEGATE_FILE` not set | Pipeline starts; no direction is active | Set the env var; run `etl-mk:validate` |
| Delegate file path is wrong | `etl-mk:start` runs but no events processed; no error logged | Use absolute paths; run `etl-mk:validate` |
| Delegate exports no matching handler | Events consumed but nothing published/written | Verify exported function names match operation types |
| All handlers return `null` | Pipeline runs, DLQ empty, nothing written | Inspect delegate return values with `KOZEN_LOG_LEVEL=DEBUG` |
| KM consumer group already at latest offset | No messages consumed | Use a new group ID or reset offset via Kafka CLI |
| MongoDB not a replica set (MK) | `MongoServerError: The $changeStream stage is only supported on replica sets` | Use a replica set or sharded cluster |

**Run `etl-mk:validate` after every configuration change** to catch missing variables before the
pipeline starts. Use `KOZEN_LOG_LEVEL=DEBUG` to trace delegate dispatch and payload flow.

---

## 10. Production readiness checklist

Before deploying `@kozen/etl-mk` to production:

**Infrastructure:**
- [ ] MongoDB is a replica set or Atlas M10+ cluster
- [ ] Kafka topic exists (or broker allows auto-creation)
- [ ] SSL enabled for all broker connections
- [ ] DLQ topic (MK) or collection (KM) exists and is monitored

**Configuration:**
- [ ] All required env vars set; `etl-mk:validate` exits 0
- [ ] Unique, descriptive `CLIENT_ID` per environment
- [ ] Unique `GROUP_ID` per logical KM consumer pipeline
- [ ] `KOZEN_STACK=prod` set for log correlation
- [ ] `KOZEN_PROJECT` set for multi-project deployments

**Delegate:**
- [ ] All handlers return `null` for events that should be skipped
- [ ] Errors are handled explicitly or allowed to propagate to DLQ
- [ ] No `console.log` — all output via `tools.logger`
- [ ] KM delegate produces idempotent documents if `writeMode=upsert` is used
- [ ] `setMessageKey()` called on every MK handler that needs ordering

**Operations:**
- [ ] Process managed by PM2, systemd, or Kubernetes (auto-restart on crash)
- [ ] DLQ depth monitored with alerting
- [ ] Log aggregation configured (Datadog, CloudWatch, etc.)
- [ ] Runbook written for DLQ reprocessing
