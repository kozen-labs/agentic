---
name: "@kozen/etl-mk — Configuration & Demo Setup"
description: >
  All environment variables for @kozen/etl-mk, organized by pipeline direction (MK and KM),
  with required/optional status, default values, and .env file examples. Includes step-by-step
  demo scaffolding for MK-only, KM-only, and bidirectional pipelines. Also covers Docker and
  PM2 deployment patterns, config validation workflow, and pre-built .env templates.
category: kozen-etl-mk
tags:
  - kozen
  - etl
  - environment-variables
  - configuration
  - demo
  - MK-pipeline
  - KM-pipeline
  - bidirectional
  - docker
  - pm2
  - validate
---

# @kozen/etl-mk — Configuration & Demo Setup

## Pipeline activation logic

A pipeline direction is activated **only** when its delegate file env var is set:

| Condition | Result |
|---|---|
| Only `KOZEN_ETL_MK_DELEGATE_FILE` is set | MK pipeline runs; KM is idle |
| Only `KOZEN_ETL_KM_DELEGATE_FILE` is set | KM pipeline runs; MK is idle |
| Both set | Both pipelines run concurrently |
| Neither set | Process starts but no pipeline runs (use `etl:validate` to detect) |

---

## MK pipeline — MongoDB → Kafka

### Required variables (when MK is active)

| Variable | CLI flag | Description |
|---|---|---|
| `KOZEN_ETL_MK_SOURCE_URI` | `--mk.source.uri` | MongoDB connection string (srv or standard) |
| `KOZEN_ETL_MK_SOURCE_DATABASE` | `--mk.source.database` | Source database name |
| `KOZEN_ETL_MK_SOURCE_COLLECTION` | `--mk.source.collection` | Source collection to watch |
| `KOZEN_ETL_MK_DESTINATION_BROKERS` | `--mk.destination.brokers` | Kafka broker list (`host1:9092,host2:9092`) |
| `KOZEN_ETL_MK_DESTINATION_TOPIC` | `--mk.destination.topic` | Target Kafka topic name |
| `KOZEN_ETL_MK_DELEGATE_FILE` | `--mk.delegateFile` | Absolute or relative path to MK delegate file |

### Optional variables (MK)

| Variable | CLI flag | Default | Description |
|---|---|---|---|
| `KOZEN_ETL_MK_DESTINATION_CLIENT_ID` | `--mk.destination.clientId` | `etl-mk` | KafkaJS client identifier shown in broker logs |
| `KOZEN_ETL_MK_DESTINATION_SSL` | `--mk.destination.ssl` | `false` | Enable TLS for Kafka connection |
| `KOZEN_ETL_MK_DELEGATE_KEY` | `--mk.delegateKey` | `etl-mk:delegate:source` | Named export key in delegate file |
| `KOZEN_ETL_MK_DLQ_TOPIC` | `--mk.dlqTopic` | `<topic>-dlq` | Dead-letter queue topic for failed publishes |

---

## KM pipeline — Kafka → MongoDB

### Required variables (when KM is active)

| Variable | CLI flag | Description |
|---|---|---|
| `KOZEN_ETL_KM_SOURCE_BROKERS` | `--km.source.brokers` | Kafka broker list (`host1:9092,host2:9092`) |
| `KOZEN_ETL_KM_SOURCE_TOPIC` | `--km.source.topic` | Kafka topic to consume |
| `KOZEN_ETL_KM_DESTINATION_URI` | `--km.destination.uri` | MongoDB connection string |
| `KOZEN_ETL_KM_DESTINATION_DATABASE` | `--km.destination.database` | Target database name |
| `KOZEN_ETL_KM_DESTINATION_COLLECTION` | `--km.destination.collection` | Target collection name |
| `KOZEN_ETL_KM_DELEGATE_FILE` | `--km.delegateFile` | Absolute or relative path to KM delegate file |

### Optional variables (KM)

| Variable | CLI flag | Default | Description |
|---|---|---|---|
| `KOZEN_ETL_KM_SOURCE_GROUP_ID` | `--km.source.groupId` | `etl-mk-group` | Kafka consumer group ID |
| `KOZEN_ETL_KM_SOURCE_CLIENT_ID` | `--km.source.clientId` | `etl-km` | KafkaJS client identifier |
| `KOZEN_ETL_KM_SOURCE_SSL` | `--km.source.ssl` | `false` | Enable TLS for Kafka connection |
| `KOZEN_ETL_KM_DELEGATE_KEY` | `--km.delegateKey` | `etl-mk:delegate:destination` | Named export key in delegate file |
| `KOZEN_ETL_KM_DESTINATION_WRITE_MODE` | `--km.writeMode` | `insert` | `insert` or `upsert` (upsert matches by `_id`) |
| `KOZEN_ETL_KM_DLQ_TOPIC` | `--km.dlqTopic` | `<topic>-dlq` | DLQ Kafka topic for failed writes |
| `KOZEN_ETL_KM_RETRY_ATTEMPTS` | `--km.retryAttempts` | `3` | Max write retry attempts before DLQ routing |
| `KOZEN_ETL_KM_RETRY_DELAY_MS` | `--km.retryDelayMs` | `1000` | Base delay (ms) between retries (linear: delay × attempt) |

---

## Common variables

| Variable | Default | Description |
|---|---|---|
| `KOZEN_ETL_DELEGATE_TYPE` | auto-detect | Force delegate loading mode: `esm` or `cjs` |
| `KOZEN_MODULE_LOAD` | — | Set to `@kozen/etl-mk` when using CLI without pre-config |
| `KOZEN_LOG_LEVEL` | `INFO` | Log verbosity: `DEBUG`, `INFO`, `WARN`, `ERROR`, `NONE` |
| `KOZEN_LOG_TYPE` | `object` | Log format: `object` (structured JSON) or `text` |
| `KOZEN_STACK` | `dev` | Environment label: `dev`, `test`, `prod` |
| `KOZEN_PROJECT` | — | Project ID for log correlation |

---

## Demo: MK pipeline (MongoDB → Kafka)

### 1. Create delegate

```javascript
// delegates/orders-mk.mjs
export async function insert(change, tools) {
  const doc = change.fullDocument;
  tools.setMessageKey(doc._id.toString());
  return { id: doc._id.toString(), status: doc.status, amount: doc.amount };
}

export async function update(change, tools) {
  const fields = change.updateDescription?.updatedFields;
  if (!fields?.status) return null;
  tools.setMessageKey(change.documentKey._id.toString());
  return { id: change.documentKey._id.toString(), status: fields.status };
}

export async function delete(change, tools) {
  tools.setMessageKey(change.documentKey._id.toString());
  return { id: change.documentKey._id.toString(), deleted: true };
}
```

### 2. Create .env file

```bash
# .env.mk-demo
KOZEN_MODULE_LOAD=@kozen/etl-mk
KOZEN_LOG_LEVEL=INFO

# MK source: MongoDB
KOZEN_ETL_MK_SOURCE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
KOZEN_ETL_MK_SOURCE_DATABASE=mydb
KOZEN_ETL_MK_SOURCE_COLLECTION=orders

# MK destination: Kafka
KOZEN_ETL_MK_DESTINATION_BROKERS=localhost:9092
KOZEN_ETL_MK_DESTINATION_TOPIC=orders.events
KOZEN_ETL_MK_DESTINATION_CLIENT_ID=demo-mk

# Delegate
KOZEN_ETL_MK_DELEGATE_FILE=./delegates/orders-mk.mjs

# DLQ (optional)
KOZEN_ETL_MK_DLQ_TOPIC=orders.events-dlq
```

### 3. Validate and start

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:validate --envFile=.env.mk-demo
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:start   --envFile=.env.mk-demo
```

---

## Demo: KM pipeline (Kafka → MongoDB)

### 1. Create delegate

```javascript
// delegates/archive-km.mjs
export async function message(msg, tools) {
  return {
    ...msg,
    archivedAt:  new Date(),
    source:      tools.collectionName,
    flow:        tools.flow,
  };
}
```

### 2. Create .env file

```bash
# .env.km-demo
KOZEN_MODULE_LOAD=@kozen/etl-mk
KOZEN_LOG_LEVEL=INFO

# KM source: Kafka
KOZEN_ETL_KM_SOURCE_BROKERS=localhost:9092
KOZEN_ETL_KM_SOURCE_TOPIC=orders.events
KOZEN_ETL_KM_SOURCE_GROUP_ID=demo-km-group

# KM destination: MongoDB
KOZEN_ETL_KM_DESTINATION_URI=mongodb+srv://user:pass@cluster.mongodb.net/
KOZEN_ETL_KM_DESTINATION_DATABASE=archive
KOZEN_ETL_KM_DESTINATION_COLLECTION=orders_archive
KOZEN_ETL_KM_DESTINATION_WRITE_MODE=insert

# Delegate
KOZEN_ETL_KM_DELEGATE_FILE=./delegates/archive-km.mjs

# Retry & DLQ
KOZEN_ETL_KM_RETRY_ATTEMPTS=3
KOZEN_ETL_KM_RETRY_DELAY_MS=1000
KOZEN_ETL_KM_DLQ_TOPIC=orders.events-dlq
```

### 3. Validate and start

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:validate --envFile=.env.km-demo
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:start   --envFile=.env.km-demo
```

---

## Demo: Bidirectional pipeline

```bash
# .env.bidirectional
KOZEN_MODULE_LOAD=@kozen/etl-mk
KOZEN_LOG_LEVEL=INFO

# MK (MongoDB → Kafka)
KOZEN_ETL_MK_SOURCE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
KOZEN_ETL_MK_SOURCE_DATABASE=production
KOZEN_ETL_MK_SOURCE_COLLECTION=orders
KOZEN_ETL_MK_DESTINATION_BROKERS=broker1:9092,broker2:9092
KOZEN_ETL_MK_DESTINATION_TOPIC=orders.events
KOZEN_ETL_MK_DELEGATE_FILE=./delegates/orders-mk.mjs

# KM (Kafka → MongoDB)
KOZEN_ETL_KM_SOURCE_BROKERS=broker1:9092,broker2:9092
KOZEN_ETL_KM_SOURCE_TOPIC=orders.events
KOZEN_ETL_KM_SOURCE_GROUP_ID=archive-consumer-group
KOZEN_ETL_KM_DESTINATION_URI=mongodb+srv://user:pass@cluster.mongodb.net/
KOZEN_ETL_KM_DESTINATION_DATABASE=archive
KOZEN_ETL_KM_DESTINATION_COLLECTION=orders_archive
KOZEN_ETL_KM_DESTINATION_WRITE_MODE=insert
KOZEN_ETL_KM_DELEGATE_FILE=./delegates/archive-km.mjs
```

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:start --envFile=.env.bidirectional
```

Or using the bundled template:
```bash
cp node_modules/@kozen/etl-mk/cfg/env.bidirectional.example .env
# Edit .env with your connection strings and delegate paths
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:start --envFile=.env
```

---

## Bundled .env templates

`@kozen/etl-mk` ships three ready-to-use templates in `cfg/`:

| File | Use case |
|---|---|
| `cfg/env.mongo-to-kafka.example` | MK pipeline only |
| `cfg/env.kafka-to-mongo.example` | KM pipeline only |
| `cfg/env.bidirectional.example` | Both directions concurrently |

Copy the relevant template, fill in credentials and delegate paths, then run.

---

## Docker deployment

```dockerfile
FROM node:18-alpine
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY delegates/ ./delegates/

CMD ["npx", "kozen", "--moduleLoad=@kozen/etl-mk", "--action=etl:start"]
```

```bash
docker run --rm \
  -e KOZEN_ETL_MK_SOURCE_URI="mongodb+srv://..." \
  -e KOZEN_ETL_MK_SOURCE_DATABASE="mydb" \
  -e KOZEN_ETL_MK_SOURCE_COLLECTION="orders" \
  -e KOZEN_ETL_MK_DESTINATION_BROKERS="broker1:9092" \
  -e KOZEN_ETL_MK_DESTINATION_TOPIC="orders.events" \
  -e KOZEN_ETL_MK_DELEGATE_FILE="/app/delegates/orders-mk.mjs" \
  my-etl-image
```

---

## PM2 deployment

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name:   'etl-mk-pipeline',
    script: 'node_modules/@kozen/engine/dist/bin/kozen.js',
    args:   '--moduleLoad=@kozen/etl-mk --action=etl:start',
    env: {
      KOZEN_ETL_MK_SOURCE_URI:              process.env.MONGO_URI,
      KOZEN_ETL_MK_SOURCE_DATABASE:         'production',
      KOZEN_ETL_MK_SOURCE_COLLECTION:       'orders',
      KOZEN_ETL_MK_DESTINATION_BROKERS:     process.env.KAFKA_BROKERS,
      KOZEN_ETL_MK_DESTINATION_TOPIC:       'orders.events',
      KOZEN_ETL_MK_DELEGATE_FILE:           '/opt/app/delegates/orders-mk.mjs',
      KOZEN_ETL_KM_SOURCE_BROKERS:          process.env.KAFKA_BROKERS,
      KOZEN_ETL_KM_SOURCE_TOPIC:            'orders.events',
      KOZEN_ETL_KM_DESTINATION_URI:         process.env.MONGO_URI,
      KOZEN_ETL_KM_DESTINATION_DATABASE:    'archive',
      KOZEN_ETL_KM_DESTINATION_COLLECTION:  'orders_archive',
      KOZEN_ETL_KM_DELEGATE_FILE:           '/opt/app/delegates/archive-km.mjs',
    }
  }]
};
```

```bash
pm2 start ecosystem.config.js
pm2 logs etl-mk-pipeline
```

---

## Validation workflow (CI/CD)

Run `etl:validate` as a pre-flight check before any deployment to catch missing env vars:

```bash
# In CI/CD pipeline
npx kozen --moduleLoad=@kozen/etl-mk --action=etl:validate
if [ $? -ne 0 ]; then
  echo "ETL configuration validation failed — aborting deployment"
  exit 1
fi
```

`etl:validate` checks:
- All required MK variables present (when MK delegate file is set)
- All required KM variables present (when KM delegate file is set)
- Delegate files are resolvable paths
- Logs each missing variable before exiting with a non-zero code
