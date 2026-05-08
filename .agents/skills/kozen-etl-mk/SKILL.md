---
name: kozen-etl-mk
description: >
  Reference skill for @kozen/etl-mk — a Kozen module that runs bi-directional ETL pipelines
  between MongoDB and Apache Kafka. Covers when and why to use this npm package, prerequisites,
  the MK (MongoDB→Kafka) and KM (Kafka→MongoDB) pipelines, the delegate pattern for transforming
  data in both directions, all CLI commands (etl-mk:start, etl-mk:validate, etl-mk:help), all environment
  variables, IoC service registrations, DLQ routing, retry logic, best practices for production,
  and how to build runnable demos using .env files and ESM delegate files.
created: 2026-05-06
updated: 2026-05-06
---

# @kozen/etl-mk — MongoDB ↔ Kafka ETL Module

## What this package does

`@kozen/etl-mk` is an npm package for the Kozen framework that runs production-grade ETL
pipelines between MongoDB and Apache Kafka. You write only the transform logic (a delegate
file); the module handles connections, change streams, producer/consumer lifecycle, dead-letter
queuing, retries, and structured logging.

**Two pipeline directions:**
- **MK** (MongoDB → Kafka): watches a MongoDB collection via change stream, transforms each
  change event in a delegate, and publishes the result to a Kafka topic.
- **KM** (Kafka → MongoDB): consumes messages from a Kafka topic, transforms each message
  in a delegate, and writes the result to a MongoDB collection with at-least-once delivery.

Both directions can run concurrently in a single process.

---

## When to use this package

Use `@kozen/etl-mk` when you need to move or replicate data between MongoDB and Kafka without
writing and maintaining pipeline boilerplate yourself.

| Scenario | Direction |
|---|---|
| Stream MongoDB document changes to downstream Kafka consumers | MK |
| Feed Kafka events into MongoDB for persistence, archiving, or analytics | KM |
| Keep a MongoDB collection in sync with an event-sourced Kafka topic | KM with `writeMode=upsert` |
| Mirror or replicate a MongoDB collection through Kafka to another MongoDB | MK + KM — two separate services |
| React to MongoDB changes and produce enriched or filtered Kafka messages | MK with transform delegate |
| Audit trail: persist all Kafka domain events into MongoDB for querying | KM |

**Use MK when:** MongoDB is your source of truth and downstream systems consume Kafka events.

**Use KM when:** Kafka is your event bus and MongoDB is the read/query store or archive.

**Use two separate services (not bidirectional in one process) when you need both directions.**
Running MK and KM as independent containers is the recommended deployment model — each service
can be scaled, restarted, or deployed independently without affecting the other. Both share the
same Docker image; what differs is the env file each container receives. See `references/configuration.md`
for the complete two-service Docker Compose pattern.

### When NOT to use this package

- You need real-time OLAP or stream processing with aggregations, joins, or windowing — use
  Apache Flink, Kafka Streams, or Atlas Stream Processing instead.
- You need Kafka Connect with a managed connector (e.g., Debezium) — `@kozen/etl-mk` is
  code-first; if you need a declarative, connector-managed pipeline, use Kafka Connect.
- You need exactly-once semantics — this module guarantees at-least-once on the KM side;
  your delegate must be idempotent if deduplication matters.
- You do not have a Kozen framework project — this module requires `@kozen/engine` as a peer.

---

## Prerequisites

Before running any pipeline, verify these conditions:

### MongoDB
- [ ] MongoDB instance is a **replica set or sharded cluster** — standalone instances do not
  support change streams (required by MK).
- [ ] The connection user has `read` on the source collection and `changeStream` privilege.
- [ ] For Atlas: M10+ cluster tier (change streams not available on M0/M2/M5).

### Kafka
- [ ] A running Kafka broker accessible from the process (local or remote).
- [ ] The target topic exists, or the broker is configured with `auto.create.topics.enable=true`.
- [ ] If SSL is required, set `KOZEN_ETL_MK_DESTINATION_SSL=true` / `KOZEN_ETL_KM_SOURCE_SSL=true`.

### Kozen project
- [ ] `@kozen/engine` is installed as a peer dependency (`npm install @kozen/engine`).
- [ ] `@kozen/etl-mk` is installed (`npm install @kozen/etl-mk`).
- [ ] Node.js ≥ 18 is running in the environment.

---

## Routing table

| Signal | Reference |
|---|---|
| EtlModule, IEtlOptions, IMongoConfig, IKafkaConfig, IMongoToKafkaConfig, IKafkaToMongoConfig, IEtlMongoToKafkaTools, IKafkaDelegate, EtlPipelineService, MongoToKafkaService, KafkaToMongoService, KafkaProducerService, KafkaConsumerService, MongoWriterService, DelegateLoaderService, EtlCLIController, delegate pattern, MK pipeline, KM pipeline, bidirectional, DLQ, retry, at-least-once, etl-mk:start, etl-mk:validate, etl-mk:help, setMessageKey, setMessageHeaders, writeMode, insert upsert, onChange, onMessage, IoC registration, etl-mk:pipeline, etl-mk:kafka-producer, etl-mk:kafka-consumer, etl-mk:mongo-writer, etl-mk:mongo-to-kafka, etl-mk:kafka-to-mongo, ChangeStreamService | `references/api.md` |
| KOZEN_ETL_MK_SOURCE_URI, KOZEN_ETL_MK_SOURCE_DATABASE, KOZEN_ETL_MK_SOURCE_COLLECTION, KOZEN_ETL_MK_DESTINATION_BROKERS, KOZEN_ETL_MK_DESTINATION_TOPIC, KOZEN_ETL_KM_SOURCE_BROKERS, KOZEN_ETL_KM_SOURCE_TOPIC, KOZEN_ETL_KM_SOURCE_GROUP_ID, KOZEN_ETL_KM_DESTINATION_URI, KOZEN_ETL_KM_DESTINATION_DATABASE, KOZEN_ETL_KM_DESTINATION_COLLECTION, KOZEN_ETL_KM_DESTINATION_WRITE_MODE, KOZEN_ETL_DELEGATE_TYPE, DLQ topic, retry attempts, retry delay, SSL, .env file, .env.mk, .env.km, demo setup, two-service deployment, separate services, shared dockerfile, env file isolation, docker-compose, etl-mk service, etl-km service, Kafka dual listeners, MongoDB replica set Docker, mongo-init, validate per service, PM2, validate config | `references/configuration.md` |
| delegate design, idempotency, at-least-once implications, error handling in delegates, Kafka message key strategy, consumer group ID, SSL production, DLQ monitoring, writeMode selection, ordering guarantees, silent failure patterns, structured logging, secrets in delegates | `references/best-practices.md` |

---

## How to use this package — the 3-step model

Every usage of `@kozen/etl-mk` follows the same three steps regardless of direction:

### Step 1 — Write a delegate file

The delegate is a `.mjs` (ESM) or `.cjs` (CommonJS) file that exports named async functions.
It contains **only your business logic** — no connection code, no Kafka SDK, no MongoDB driver.

```javascript
// delegates/my-transform.mjs
export async function insert(change, tools) {   // MK: called for each MongoDB insert
  return { id: change.fullDocument._id.toString(), ...change.fullDocument };
}

export async function message(msg, tools) {     // KM: called for each Kafka message
  return { ...msg, savedAt: new Date() };
}
```

### Step 2 — Configure via environment variables (or CLI flags)

```bash
# .env — minimum required for MK
KOZEN_ETL_MK_SOURCE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
KOZEN_ETL_MK_SOURCE_DATABASE=mydb
KOZEN_ETL_MK_SOURCE_COLLECTION=orders
KOZEN_ETL_MK_DESTINATION_BROKERS=localhost:9092
KOZEN_ETL_MK_DESTINATION_TOPIC=orders.events
KOZEN_ETL_MK_DELEGATE_FILE=./delegates/my-transform.mjs
```

### Step 3 — Validate, then start

```bash
# Always validate first
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env

# Start the pipeline (long-running process)
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env
```

The process stays alive, listening for events. Send `SIGINT` (Ctrl+C) to stop gracefully.

---

## Quick start — runnable examples

### MongoDB → Kafka (MK)

```bash
npm install @kozen/etl-mk

# delegates/orders.mjs
cat > delegates/orders.mjs << 'EOF'
export async function insert(change, tools) {
  const doc = change.fullDocument;
  tools.setMessageKey(doc._id.toString());
  return { id: doc._id.toString(), status: doc.status, amount: doc.amount };
}
export async function update(change, tools) {
  const fields = change.updateDescription?.updatedFields;
  if (!fields?.status) return null; // skip events we don't care about
  tools.setMessageKey(change.documentKey._id.toString());
  return { id: change.documentKey._id.toString(), status: fields.status };
}
EOF

export KOZEN_ETL_MK_SOURCE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
export KOZEN_ETL_MK_SOURCE_DATABASE=mydb
export KOZEN_ETL_MK_SOURCE_COLLECTION=orders
export KOZEN_ETL_MK_DESTINATION_BROKERS=localhost:9092
export KOZEN_ETL_MK_DESTINATION_TOPIC=orders.events
export KOZEN_ETL_MK_DELEGATE_FILE=./delegates/orders.mjs

npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start
```

### Kafka → MongoDB (KM)

```bash
cat > delegates/archive.mjs << 'EOF'
export async function message(msg, tools) {
  return { ...msg, archivedAt: new Date(), source: tools.collectionName };
}
EOF

export KOZEN_ETL_KM_SOURCE_BROKERS=localhost:9092
export KOZEN_ETL_KM_SOURCE_TOPIC=orders.events
export KOZEN_ETL_KM_DESTINATION_URI=mongodb+srv://user:pass@cluster.mongodb.net/
export KOZEN_ETL_KM_DESTINATION_DATABASE=archive
export KOZEN_ETL_KM_DESTINATION_COLLECTION=orders_archive
export KOZEN_ETL_KM_DELEGATE_FILE=./delegates/archive.mjs

npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start
```

### Validate configuration without connecting

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate
# Exit 0 = valid. Non-zero = logs each missing variable, then fails.
```

---

## When to invoke this skill (AI usage)

Invoke this skill when the user is:
- Scaffolding a new MK or KM demo or pipeline from scratch
- Asking what delegate exports to write for a given use case
- Troubleshooting a missing env var, delegate not loading, or pipeline not activating
- Designing the writeMode, DLQ, or retry strategy for a KM pipeline
- Embedding `@kozen/etl-mk` programmatically in a Kozen application
- Asking about best practices for production ETL with this module

---

## Do & Don't

**Do:**
- Always run `etl-mk:validate` before `etl-mk:start` in CI/CD — it exits non-zero on missing vars.
- Set `KOZEN_ETL_MK_DELEGATE_FILE` to activate MK; omit it entirely to disable MK silently.
- Return `null` or `undefined` from any delegate handler to skip that event without error.
- Use `.mjs` for ESM delegates (recommended) and `.cjs` for CommonJS — auto-detected by extension.
- Set `writeMode=upsert` and make delegates idempotent on KM — at-least-once means duplicates will occur.
- Always set a meaningful `KOZEN_ETL_MK_DESTINATION_CLIENT_ID` / `KOZEN_ETL_KM_SOURCE_CLIENT_ID` per environment.
- Use a unique `KOZEN_ETL_KM_SOURCE_GROUP_ID` per logical consumer group — sharing one group between pipelines causes missed messages.
- Configure DLQ topics in production — failed messages vanish silently without them.
- Use `tools.logger` (not `console.log`) inside delegates for structured, traceable output.

**Don't:**
- Never import KafkaJS, MongoClient, or other infrastructure SDKs directly in delegate files — use `tools`.
- Never use synchronous CPU-blocking code in delegates — all processing is event-loop-based.
- Never assume exactly-once delivery on KM — design your target collection schema and delegate for idempotency.
- Never omit `setMessageKey()` on MK when ordering matters — without a key, Kafka distributes messages across partitions randomly.
- Never share a consumer group ID across multiple running pipelines consuming the same topic — they will split the partition load unpredictably.
- Never run without a DLQ topic in production — an error without DLQ drops the message permanently after max retries.
- Never skip `etl-mk:validate` when migrating between environments — missing vars produce silent, hard-to-diagnose failures.
