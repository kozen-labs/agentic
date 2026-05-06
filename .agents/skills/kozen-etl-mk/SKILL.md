---
name: kozen-etl-mk
description: >
  Reference skill for @kozen/etl-mk — a Kozen module that runs bi-directional ETL pipelines
  between MongoDB and Apache Kafka. Covers the MongoDB→Kafka (MK) and Kafka→MongoDB (KM)
  pipelines, the delegate pattern for transforming data in both directions, all CLI commands
  (etl:start, etl:validate, etl:help), all environment variables, IoC service registrations,
  DLQ routing, retry logic, and how to build runnable demos using .env files and ESM delegate
  files. Use this skill to scaffold a working demo from scratch.
created: 2026-05-06
updated: 2026-05-06
---

# @kozen/etl-mk — MongoDB ↔ Kafka ETL Module

`@kozen/etl-mk` extends the Kozen framework with a bi-directional ETL pipeline between
MongoDB and Apache Kafka. A single process can run either direction, both concurrently, or
validate configuration without connecting. All business logic lives in user-supplied delegate
files; the module handles connection management, error handling, DLQ routing, and retries.

---

## Routing table

| Signal | Reference |
|---|---|
| EtlModule, IEtlOptions, IMongoConfig, IKafkaConfig, IMongoToKafkaConfig, IKafkaToMongoConfig, IEtlMongoToKafkaTools, IKafkaDelegate, EtlPipelineService, MongoToKafkaService, KafkaToMongoService, KafkaProducerService, KafkaConsumerService, MongoWriterService, DelegateLoaderService, EtlCLIController, delegate pattern, MK pipeline, KM pipeline, bidirectional, DLQ, retry, at-least-once, etl:start, etl:validate, etl:help, setMessageKey, setMessageHeaders, writeMode, insert upsert, onChange, onMessage, IoC registration, etl-mk:pipeline, etl-mk:kafka-producer, etl-mk:kafka-consumer, etl-mk:mongo-writer, etl-mk:mongo-to-kafka, etl-mk:kafka-to-mongo, ChangeStreamService | `references/api.md` |
| KOZEN_ETL_MK_SOURCE_URI, KOZEN_ETL_MK_SOURCE_DATABASE, KOZEN_ETL_MK_SOURCE_COLLECTION, KOZEN_ETL_MK_DESTINATION_BROKERS, KOZEN_ETL_MK_DESTINATION_TOPIC, KOZEN_ETL_KM_SOURCE_BROKERS, KOZEN_ETL_KM_SOURCE_TOPIC, KOZEN_ETL_KM_SOURCE_GROUP_ID, KOZEN_ETL_KM_DESTINATION_URI, KOZEN_ETL_KM_DESTINATION_DATABASE, KOZEN_ETL_KM_DESTINATION_COLLECTION, KOZEN_ETL_KM_DESTINATION_WRITE_MODE, KOZEN_ETL_DELEGATE_TYPE, DLQ topic, retry attempts, retry delay, SSL, .env file, demo setup, docker demo, PM2 demo, validate config | `references/configuration.md` |

---

## When to use this skill

- Scaffolding a working MK or KM demo from scratch (delegate + .env + start command)
- Connecting MongoDB change streams to Kafka topics without writing pipeline boilerplate
- Consuming Kafka messages and persisting them into MongoDB with at-least-once delivery
- Running both directions concurrently as a single Kozen process
- Validating environment variables before starting a pipeline in CI/CD
- Understanding the delegate interface to author MK or KM transform handlers

---

## Quick start

```bash
npm install @kozen/etl-mk
```

### MongoDB → Kafka (MK)

```bash
# 1. Create a delegate file
cat > delegates/orders.mjs << 'EOF'
export async function insert(change, tools) {
  const doc = change.fullDocument;
  tools.setMessageKey(doc._id.toString());
  return { id: doc._id.toString(), status: doc.status, amount: doc.amount };
}
export async function update(change, tools) {
  if (!change.updateDescription?.updatedFields?.status) return null;
  return { status: change.updateDescription.updatedFields.status };
}
EOF

# 2. Set environment variables
export KOZEN_ETL_MK_SOURCE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
export KOZEN_ETL_MK_SOURCE_DATABASE=mydb
export KOZEN_ETL_MK_SOURCE_COLLECTION=orders
export KOZEN_ETL_MK_DESTINATION_BROKERS=broker1:9092,broker2:9092
export KOZEN_ETL_MK_DESTINATION_TOPIC=orders.events
export KOZEN_ETL_MK_DELEGATE_FILE=./delegates/orders.mjs

# 3. Start the pipeline
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:start
```

### Kafka → MongoDB (KM)

```bash
cat > delegates/archive.mjs << 'EOF'
export async function message(msg, tools) {
  return { ...msg, archivedAt: new Date(), source: tools.collectionName };
}
EOF

export KOZEN_ETL_KM_SOURCE_BROKERS=broker1:9092,broker2:9092
export KOZEN_ETL_KM_SOURCE_TOPIC=orders.events
export KOZEN_ETL_KM_DESTINATION_URI=mongodb+srv://user:pass@cluster.mongodb.net/
export KOZEN_ETL_KM_DESTINATION_DATABASE=archive
export KOZEN_ETL_KM_DESTINATION_COLLECTION=orders_archive
export KOZEN_ETL_KM_DELEGATE_FILE=./delegates/archive.mjs

npx kozen --moduleLoad=@kozen/etl-mk --action=etl:start
```

### Validate configuration without connecting

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:validate
```

---

## Do & Don't

**Do:**
- Activate a pipeline direction by setting its `DELEGATE_FILE` env var; omit the var to disable that direction.
- Return `null` or `undefined` from a delegate handler to skip publishing or writing.
- Use `.mjs` for ESM delegates and `.cjs` for CommonJS — the module auto-detects by extension.
- Set `KOZEN_ETL_KM_DESTINATION_WRITE_MODE=upsert` when idempotent writes are required for KM.
- Run `etl:validate` in CI/CD before deploying to catch missing environment variables early.
- Set `KOZEN_ETL_KM_RETRY_ATTEMPTS` and `KOZEN_ETL_KM_RETRY_DELAY_MS` for resilient KM pipelines.
- Use DLQ topics to capture failed messages for later reprocessing instead of discarding them.

**Don't:**
- Never import concrete service classes (`KafkaProducerService`, etc.) in delegates — receive them via `tools`.
- Never use `console.log` in delegates — use `tools.logger` to stay within the structured logging pipeline.
- Never restart the pipeline without a delegate file present — the pipeline will start with neither direction active.
- Don't block the delegate handler with synchronous CPU work — all processing is event-loop-based.
- Don't skip `etl:validate` when migrating to a new environment — env var mismatches cause silent failures.
