---
name: "@kozen/etl-mk — Configuration & Demo Setup"
description: >
  All environment variables for @kozen/etl-mk organized by pipeline direction (MK and KM),
  with required/optional status and default values. Covers the recommended two-service Docker
  Compose deployment (shared Dockerfile, separate .env.mk and .env.km per service, why env
  file isolation prevents accidental bidirectional activation), full docker-compose.yml with
  Kafka dual-listener, MongoDB replica set init, etl-mk and etl-km as independent containers,
  per-service validation, troubleshooting table, PM2 two-process pattern, and CI/CD validation
  workflow. Also covers .env file demos for MK-only, KM-only, and bundled templates.
category: kozen-etl-mk
tags:
  - kozen
  - etl
  - environment-variables
  - configuration
  - two-service-deployment
  - shared-dockerfile
  - env-file-isolation
  - docker-compose
  - etl-mk-service
  - etl-km-service
  - kafka-dual-listener
  - mongodb-replica-set
  - mongo-init
  - pm2
  - validate
  - demo
---

# @kozen/etl-mk — Configuration & Demo Setup

## Pipeline activation logic

A pipeline direction is activated **only** when its delegate file env var is set:

| Condition | Result |
|---|---|
| Only `KOZEN_ETL_MK_DELEGATE_FILE` is set | MK pipeline runs; KM is idle |
| Only `KOZEN_ETL_KM_DELEGATE_FILE` is set | KM pipeline runs; MK is idle |
| Both set | Both pipelines run concurrently |
| Neither set | Process starts but no pipeline runs (use `etl-mk:validate` to detect) |

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
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env.mk-demo
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start   --envFile=.env.mk-demo
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
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env.km-demo
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start   --envFile=.env.km-demo
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
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env.bidirectional
```

Or using the bundled template:
```bash
cp node_modules/@kozen/etl-mk/cfg/env.bidirectional.example .env
# Edit .env with your connection strings and delegate paths
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env
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

## Two-service deployment (recommended)

`@kozen/etl-mk` runs as two independent processes: a **producer** (MongoDB → Kafka) and a
**consumer** (Kafka → MongoDB). **The module name never changes** — only the env file tells
each process which role to play.

| Process | Kafka role | Env file | Direction |
|---|---|---|---|
| `etl-mk` | **Producer** — publishes to Kafka | `.env.mk` | MongoDB → Kafka |
| `etl-km` | **Consumer** — reads from Kafka | `.env.km` | Kafka → MongoDB |

```bash
# Producer: MongoDB is source, Kafka is destination
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env.mk

# Consumer: Kafka is source, MongoDB is destination
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env.km
```

The `--moduleLoad` flag is always `@kozen/etl-mk` for both processes. The env file
activates the pipeline direction — `.env.mk` carries the MongoDB-source variables that
activate the producer pipeline; `.env.km` carries the Kafka-source variables that activate
the consumer pipeline.

### Why two processes instead of one bidirectional process?

The pipeline activates a direction when the corresponding `DELEGATE_FILE` env var is set.
If both `KOZEN_ETL_MK_DELEGATE_FILE` and `KOZEN_ETL_KM_DELEGATE_FILE` are present in the
same process, both pipelines start — making independent scaling, restarts, and failure
isolation impossible.

In practice, producer and consumer have different throughput profiles:

- A spike in MongoDB writes increases **producer** load → scale producer instances.
- A growing Kafka backlog increases **consumer** load → scale consumer instances.

Running them separately lets you scale each independently. Kafka handles partition
distribution automatically when you run multiple consumer instances with the same
`KOZEN_ETL_KM_SOURCE_GROUP_ID`.

### Manual deployment (baseline)

The simplest starting point — run both processes directly on the host:

```bash
# Validate and start the producer (terminal 1)
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env.mk
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start   --envFile=.env.mk

# Validate and start the consumer (terminal 2)
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env.km
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start   --envFile=.env.km
```

To add more consumer instances, start additional processes with the same `.env.km` — Kafka
assigns partitions automatically across all instances in the same consumer group:

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env.km  # instance 2
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env.km  # instance 3
```

Docker Compose formalises this model with health checks, restart policies, and one-command
scaling — covered in the section below.

### Two-service Docker Compose deployment

### File layout

```
etl-orders/
├── docker-compose.yml    ← full stack (Zookeeper, Kafka, MongoDB, two ETL services)
├── Dockerfile            ← shared image for both ETL services
├── package.json          ← runtime dependencies (no devDependencies)
├── .env.mk               ← ONLY MK variables — etl-mk service — Producer
├── .env.km               ← ONLY KM variables — etl-km service — Consumer
└── delegates/
    ├── orders.mjs        ← MK delegate: MongoDB change → Kafka payload
    └── archive.mjs       ← KM delegate: Kafka message → MongoDB document
```

Both `.env.mk` and `.env.km` must be added to `.gitignore`.

---

### package.json (runtime only)

```json
{
  "name": "orders-etl",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@kozen/engine": "^1.1.15",
    "@kozen/etl-mk": "^1.0.0"
  }
}
```

---

### Dockerfile (shared by both services)

```dockerfile
FROM node:18-alpine
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY delegates/ ./delegates/

CMD ["npx", "kozen", "--moduleLoad=@kozen/etl-mk", "--action=etl-mk:start"]
```

The `command:` in `docker-compose.yml` can override the CMD if needed. Delegates are baked
into the image. To update them without rebuilding, mount as a volume instead:
```yaml
volumes:
  - ./delegates:/app/delegates
```

---

### .env.mk — MK service only

```bash
# .env.mk — MongoDB → Kafka pipeline only. DO NOT commit this file.
KOZEN_ETL_MK_SOURCE_URI=mongodb://mongodb:27017/mydb?replicaSet=rs0
KOZEN_ETL_MK_SOURCE_DATABASE=mydb
KOZEN_ETL_MK_SOURCE_COLLECTION=orders
KOZEN_ETL_MK_DESTINATION_BROKERS=kafka:9092
KOZEN_ETL_MK_DESTINATION_TOPIC=orders.events
KOZEN_ETL_MK_DELEGATE_FILE=/app/delegates/orders.mjs
# KOZEN_ETL_MK_DLQ_TOPIC=orders.events-dlq

KOZEN_LOG_LEVEL=INFO
KOZEN_LOG_TYPE=object
```

### .env.km — KM service only

```bash
# .env.km — Kafka → MongoDB pipeline only. DO NOT commit this file.
KOZEN_ETL_KM_SOURCE_BROKERS=kafka:9092
KOZEN_ETL_KM_SOURCE_TOPIC=orders.events
KOZEN_ETL_KM_SOURCE_GROUP_ID=etl-orders-group
KOZEN_ETL_KM_DESTINATION_URI=mongodb://mongodb:27017/mydb?replicaSet=rs0
KOZEN_ETL_KM_DESTINATION_DATABASE=mydb
KOZEN_ETL_KM_DESTINATION_COLLECTION=orders_archive
KOZEN_ETL_KM_DELEGATE_FILE=/app/delegates/archive.mjs
# KOZEN_ETL_KM_DESTINATION_WRITE_MODE=insert
# KOZEN_ETL_KM_DLQ_TOPIC=orders.events-dlq
# KOZEN_ETL_KM_RETRY_ATTEMPTS=3
# KOZEN_ETL_KM_RETRY_DELAY_MS=1000

KOZEN_LOG_LEVEL=INFO
KOZEN_LOG_TYPE=object
```

**Critical:** `kafka:9092` is the internal container-to-container listener. Use
`localhost:9092` only from the host machine. See the Kafka dual-listener setup below.

---

### docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:

  # ── Kafka (KRaft — no Zookeeper) ──────────────────────────────────────────────
  # Two listeners:
  #   PLAINTEXT://localhost:9092       → host machine access
  #   PLAINTEXT_INTERNAL://kafka:9092 → container-to-container access
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka_data:/var/lib/kafka/data
    networks:
      - etl-net
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 15s
      timeout: 10s
      retries: 10

  # ── Kafka UI ──────────────────────────────────────────────────────────────────
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8181:8080"
    environment:
      DYNAMIC_CONFIG_ENABLED: "true"
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    networks:
      - etl-net
    depends_on:
      kafka:
        condition: service_healthy

  # ── MongoDB (single-node replica set — required for change streams) ────────────
  mongodb:
    image: mongo:7
    hostname: mongodb
    container_name: mongodb
    ports:
      - "27017:27017"
    command: ["--replSet", "rs0", "--bind_ip_all"]
    volumes:
      - mongodb_data:/data/db
    networks:
      - etl-net
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh --quiet
      interval: 10s
      timeout: 5s
      retries: 10

  # ── Replica set initialisation (runs once, exits) ─────────────────────────────
  mongo-init:
    image: mongo:7
    container_name: mongo-init
    networks:
      - etl-net
    depends_on:
      mongodb:
        condition: service_healthy
    entrypoint: >
      mongosh --host mongodb:27017 --eval
      "try { rs.status() } catch(e) { rs.initiate({ _id: 'rs0', members: [{ _id: 0, host: 'mongodb:27017' }] }) }"
    restart: "no"

  # ── MK service: MongoDB → Kafka ───────────────────────────────────────────────
  etl-mk:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: etl-mk
    command: ["npx", "kozen", "--moduleLoad=@kozen/etl-mk", "--action=etl-mk:start"]
    env_file: .env.mk
    depends_on:
      kafka:
        condition: service_healthy
    restart: on-failure
    networks:
      - etl-net
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # ── KM service: Kafka → MongoDB ───────────────────────────────────────────────
  etl-km:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: etl-km
    command: ["npx", "kozen", "--moduleLoad=@kozen/etl-mk", "--action=etl-mk:start"]
    env_file: .env.km
    depends_on:
      kafka:
        condition: service_healthy
    restart: on-failure
    networks:
      - etl-net
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  kafka_data:

networks:
  etl-net:
    driver: bridge

```

**Kafka dual-listener note:** ETL containers use `kafka:9092` (internal listener). Host
tools (mongosh, local scripts) use `localhost:9092`. Without the dual-listener setup,
Kafka's metadata response sends `localhost:9092` to containers, causing connection failures.

---

### Build and operate

```bash
# Build the shared image once (used by both services)
docker compose build etl-mk

# Start the full stack
docker compose up -d

# Validate each service before starting (in validate mode)
docker compose run --rm etl-mk npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate
docker compose run --rm etl-km npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate

# Follow logs — MK only
docker compose logs -f etl-mk

# Follow logs — KM only
docker compose logs -f etl-km

# Follow both ETL services together
docker compose logs -f etl-mk etl-km

# Restart only the failing service (the other keeps running)
docker compose restart etl-mk
docker compose restart etl-km

# Verify each container has ONLY its own pipeline variables
docker compose run --rm etl-mk printenv | grep KOZEN
docker compose run --rm etl-km printenv | grep KOZEN
```

Expected `docker compose ps` output when healthy:

```
NAME          STATUS          PORTS
zookeeper     Up (healthy)    0.0.0.0:2181->2181/tcp
kafka         Up (healthy)    0.0.0.0:9092->9092/tcp
kafka-ui      Up              0.0.0.0:8181->8080/tcp
mongodb       Up (healthy)    0.0.0.0:27017->27017/tcp
mongo-init    Exited (0)
etl-mk        Up
etl-km        Up
```

---

### End-to-end test

```bash
# Insert a document from the host to trigger the MK pipeline
mongosh "mongodb://localhost:27017/mydb?replicaSet=rs0" --eval '
  db.orders.insertOne({
    customerId: "cust-001",
    total: 99.90,
    status: "confirmed",
    createdAt: new Date()
  })
'

# Watch both pipelines process it
docker compose logs -f etl-mk etl-km

# Confirm the document was archived
mongosh "mongodb://localhost:27017/mydb?replicaSet=rs0" --eval '
  printjson(db.orders_archive.findOne())
'
```

---

### Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Service exits: "No delegates defined" | `DELEGATE_FILE` missing from env file | Check `.env.mk` / `.env.km`; run `printenv \| grep KOZEN` inside the container |
| Both pipelines start in one container | Both `MK_*` and `KM_*` vars in same env file | Split into `.env.mk` and `.env.km`; verify with `printenv \| grep KOZEN` |
| Kafka connection refused: `localhost:9092` | ETL container resolves `localhost` to itself | Set brokers to `kafka:9092` in env files |
| `not primary and secondaryOk=false` | Replica set not initialized | Check `docker compose logs mongo-init`; run `rs.initiate()` manually if needed |
| `not a replica set` change stream error | MongoDB started without `--replSet` | Add `command: ["--replSet", "rs0", ...]` to the mongodb service |
| KM consumer never receives messages | Wrong group ID or topic name | Verify in Kafka UI at `http://localhost:8181` → Topics → Consumer Groups |
| One service fails, other keeps running | Independent services — expected behavior | Inspect failing service with `docker compose logs etl-mk` or `etl-km`; restart only that one |

---

## PM2 deployment (two processes)

The same two-service pattern applies with PM2: one app entry per pipeline direction.

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name:   'etl-mk',
      script: 'node_modules/@kozen/engine/dist/bin/kozen.js',
      args:   '--moduleLoad=@kozen/etl-mk --action=etl-mk:start',
      env_file: '.env.mk'   // only MK variables
    },
    {
      name:   'etl-km',
      script: 'node_modules/@kozen/engine/dist/bin/kozen.js',
      args:   '--moduleLoad=@kozen/etl-mk --action=etl-mk:start',
      env_file: '.env.km'   // only KM variables
    }
  ]
};
```

```bash
pm2 start ecosystem.config.js
pm2 logs etl-mk
pm2 logs etl-km
pm2 restart etl-mk   # restart only MK without touching KM
```

---

## Validation workflow (CI/CD)

Run `etl-mk:validate` as a pre-flight check for each service before deployment:

```bash
# Validate MK pipeline
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env.mk
if [ $? -ne 0 ]; then
  echo "MK configuration invalid — aborting deployment"
  exit 1
fi

# Validate KM pipeline
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate --envFile=.env.km
if [ $? -ne 0 ]; then
  echo "KM configuration invalid — aborting deployment"
  exit 1
fi
```

`etl-mk:validate` checks:
- All required variables present for each active pipeline direction
- Delegate file paths are resolvable
- Logs each missing variable before exiting with a non-zero code
