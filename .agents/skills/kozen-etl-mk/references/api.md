---
name: "@kozen/etl-mk — Public API"
description: >
  Full API reference for @kozen/etl-mk: all exported types and classes, IEtlOptions,
  IMongoConfig, IKafkaConfig, IMongoToKafkaConfig, IKafkaToMongoConfig, IEtlMongoToKafkaTools,
  IKafkaDelegate, EtlPipelineService, MongoToKafkaService (extends ChangeStreamService),
  KafkaToMongoService, KafkaProducerService, KafkaConsumerService, MongoWriterService,
  DelegateLoaderService, EtlCLIController, the full delegate pattern for both pipeline
  directions, IoC dependency map, at-least-once delivery mechanics, DLQ routing, retry logic,
  and programmatic embedding patterns.
category: kozen-etl-mk
tags:
  - kozen
  - etl
  - mongodb
  - kafka
  - change-stream
  - delegate
  - IEtlOptions
  - MK-pipeline
  - KM-pipeline
  - DLQ
  - retry
  - at-least-once
---

# @kozen/etl-mk — Public API

## All public exports

```typescript
import {
  EtlModule,               // KzModule subclass — Kozen module entry point
  IEtlOptions,             // Top-level pipeline configuration
  IMongoConfig,            // MongoDB connection + namespace config
  IKafkaConfig,            // Kafka brokers + topic config
  IMongoToKafkaConfig,     // MK pipeline full configuration
  IKafkaToMongoConfig,     // KM pipeline full configuration
  IEtlMongoToKafkaTools,   // Tools injected into MK delegates
  IKafkaDelegate,          // KM delegate handler interface
  EtlPipelineService,      // Orchestrator: runs MK and KM concurrently
  MongoToKafkaService,     // MK pipeline (extends ChangeStreamService)
  KafkaToMongoService,     // KM pipeline
  KafkaProducerService,    // KafkaJS producer wrapper
  KafkaConsumerService,    // KafkaJS consumer wrapper
  MongoWriterService,      // MongoDB insert / upsert writer
  DelegateLoaderService,   // ESM/CJS delegate file loader
  EtlCLIController,        // CLI dispatcher (rarely imported directly)
} from '@kozen/etl-mk';
```

---

## Configuration interfaces

### IEtlOptions — top-level config

```typescript
interface IEtlOptions {
  flow?: string;              // Distributed tracing ID injected into logs
  mk?:  IMongoToKafkaConfig; // MongoDB → Kafka pipeline config (omit to disable)
  km?:  IKafkaToMongoConfig; // Kafka → MongoDB pipeline config (omit to disable)
}
```

### IMongoConfig — MongoDB endpoint

```typescript
interface IMongoConfig {
  uri:        string; // Full MongoDB connection string (srv or standard)
  database:   string; // Database name
  collection: string; // Collection name
}
```

### IKafkaConfig — Kafka endpoint

```typescript
interface IKafkaConfig {
  brokers:   string;   // Comma-separated broker list: "host1:9092,host2:9092"
  topic:     string;   // Kafka topic name
  groupId?:  string;   // Consumer group ID (KM only; default: 'etl-mk-group')
  clientId?: string;   // KafkaJS client ID for broker logs (default: 'etl-mk' or 'etl-km')
  ssl?:      boolean;  // Enable TLS for broker connection (default: false)
}
```

### IMongoToKafkaConfig — MK full config

```typescript
interface IMongoToKafkaConfig {
  source:       IMongoConfig; // MongoDB change stream source
  destination:  IKafkaConfig; // Kafka publish target
  delegate?:    string;       // Path to delegate file (required to activate MK)
  delegateKey?: string;       // Named export key in delegate file (default: 'etl-mk:delegate:source')
  dlqTopic?:    string;       // DLQ topic name (default: '<topic>-dlq')
}
```

### IKafkaToMongoConfig — KM full config

```typescript
interface IKafkaToMongoConfig {
  source:          IKafkaConfig;    // Kafka consumer source
  destination:     IMongoConfig;    // MongoDB write target
  delegate?:       string;          // Path to delegate file (required to activate KM)
  delegateKey?:    string;          // Named export key (default: 'etl-mk:delegate:destination')
  writeMode?:      'insert'|'upsert'; // Write strategy (default: 'insert')
  dlqTopic?:       string;          // DLQ topic (default: '<topic>-dlq')
  retryAttempts?:  number;          // Max retry count on write failure (default: 3)
  retryDelayMs?:   number;          // Base delay between retries ms (default: 1000)
}
```

---

## Delegate interfaces

### IEtlMongoToKafkaTools — MK delegate tools

Extends `ITriggerTools` from `@kozen/trigger`, adding two Kafka-specific methods:

```typescript
interface IEtlMongoToKafkaTools extends ITriggerTools {
  /** Override the Kafka message key (default: document _id). */
  setMessageKey(key: string): void;

  /** Set custom Kafka message headers. */
  setMessageHeaders(headers: Record<string, string>): void;
}
```

All `ITriggerTools` properties are also available:
```typescript
// From ITriggerTools:
assistant?: IIoC;    // Kozen IoC container (resolve other services)
logger?:    ILogger; // Structured logger
flow?:      string;  // Tracing flow ID
```

### IKafkaDelegate — KM delegate shape

```typescript
interface IKafkaDelegate {
  /** Called for every consumed Kafka message. Return document or null to skip. */
  message?(msg: unknown, tools: IKafkaToMongoTools): Promise<unknown>;

  /** Catch-all handler (runs if no specific handler matched). */
  on?(msg: unknown, tools: IKafkaToMongoTools): Promise<unknown>;

  /** Alias for on (lowest priority). */
  default?(msg: unknown, tools: IKafkaToMongoTools): Promise<unknown>;
}
```

KM delegate tools (`IKafkaToMongoTools`):
```typescript
{
  assistant?:     IIoC;     // Kozen IoC container
  logger?:        ILogger;  // Structured logger
  flow?:          string;   // Tracing flow ID
  db?:            Db;       // Active MongoDB Db handle
  collection?:    Collection;  // Active MongoDB Collection handle
  dbName?:        string;   // Destination database name
  collectionName?:string;   // Destination collection name
}
```

---

## MK delegate — MongoDB → Kafka

Handler dispatch order for each change event:
1. Named export matching the operation (`insert`, `update`, `replace`, `delete`, `drop`, `rename`, `invalidate`)
2. `on` export (catch-all, runs after specific handler)
3. `default` export (lowest priority)
4. If no handler returns a value → message is skipped (not published)

```javascript
// delegates/orders.mjs — full MK delegate example
export async function insert(change, tools) {
  const doc = change.fullDocument;
  tools.setMessageKey(doc._id.toString());
  tools.setMessageHeaders({ source: 'orders-service', version: '1' });
  return {
    id:     doc._id.toString(),
    status: doc.status,
    amount: doc.amount,
    ts:     new Date().toISOString()
  };
}

export async function update(change, tools) {
  const updated = change.updateDescription?.updatedFields;
  if (!updated?.status) return null; // skip — only care about status changes
  tools.setMessageKey(change.documentKey._id.toString());
  return { id: change.documentKey._id.toString(), status: updated.status };
}

export async function delete(change, tools) {
  tools.setMessageKey(change.documentKey._id.toString());
  return { id: change.documentKey._id.toString(), deleted: true };
}

export async function on(change, tools) {
  // Runs after any specific handler — use for side effects (logging, metrics)
  tools.logger?.info({ op: change.operationType, flow: tools.flow }, 'change processed');
}
```

---

## KM delegate — Kafka → MongoDB

Handler dispatch order for each Kafka message:
1. `message` export
2. `on` export (catch-all)
3. `default` export
4. Return value becomes the MongoDB document — `null`/`undefined` skips the write

```javascript
// delegates/archive.mjs — minimal KM delegate
export async function message(msg, tools) {
  return {
    ...msg,
    archivedAt:  new Date(),
    source:      tools.collectionName,
    flow:        tools.flow,
  };
}
```

```javascript
// delegates/enriched.mjs — KM delegate with IoC usage
export async function message(msg, tools) {
  // Resolve @kozen/secret to decrypt a field before storing
  const secretMgr = await tools.assistant?.resolve('secret:manager');
  const apiKey = await secretMgr?.resolve('ENCRYPTION_KEY');

  return {
    raw:      msg,
    decrypted: decrypt(msg.payload, apiKey),
    savedAt:  new Date()
  };
}
```

---

## Service classes

### EtlPipelineService

Registered in IoC as `etl-mk:pipeline`. Orchestrates MK and KM concurrently.

**`start(options: IEtlOptions): Promise<void>`**
- Runs `MongoToKafkaService.start()` and `KafkaToMongoService.start()` concurrently via `Promise.all`
- Skips a direction if its `delegate` file path is absent from `options`

---

### MongoToKafkaService (extends ChangeStreamService)

Registered as `etl-mk:mongo-to-kafka`. Inherits MongoDB change stream lifecycle from
`@kozen/trigger`'s `ChangeStreamService`.

| Method | Signature | Description |
|---|---|---|
| `start(options)` | `(IEtlOptions) => Promise<void>` | Connects KafkaProducerService, then delegates to ChangeStreamService |
| `onChange(change, delegate?, tools?)` | `(ChangeStreamDocument, ...) => Promise<void>` | Dispatches change event to delegate; publishes return value to Kafka |
| `stop()` | `() => Promise<void>` | Disconnects Kafka producer and closes change stream |

**DLQ routing (MK):**
When the delegate throws or `KafkaProducerService.publish()` fails:
```json
{
  "originalPayload": "<serialized change document>",
  "error": "<error message>",
  "flow": "<flow ID>",
  "ts": "<ISO timestamp>"
}
```
Published to `dlqTopic` (default: `<topic>-dlq`).

---

### KafkaToMongoService

Registered as `etl-mk:kafka-to-mongo`. Manages the consumer → writer → commit loop.

| Method | Signature | Description |
|---|---|---|
| `start(options)` | `(IEtlOptions) => Promise<void>` | Connects consumer and writer, loads delegate, starts message loop |
| `onMessage(payload, delegate, tools, km)` | `(unknown, IKafkaDelegate, ..., IKafkaToMongoConfig) => Promise<void>` | Retry loop: dispatches message → writes to MongoDB → commits offset |

**At-least-once delivery:** Kafka offset committed only after successful MongoDB write or DLQ routing.
Offset is **never** committed on unhandled exception — message will be reprocessed after restart.

**Retry logic:**
- Up to `retryAttempts` (default: 3) attempts
- Delay between attempts: `retryDelayMs × attempt` ms (linear backoff)
- On max retries exceeded → publishes to DLQ; offset committed; processing continues

**DLQ routing (KM):**
```json
{
  "originalMessage": "<serialized Kafka message>",
  "error": "<error message>",
  "flow": "<flow ID>",
  "ts": "<ISO timestamp>"
}
```
Written to `<destinationDatabase>.<topic>-dlq` MongoDB collection.

---

### KafkaProducerService

Registered as `etl-mk:kafka-producer`.

| Method | Signature | Description |
|---|---|---|
| `connect(brokers, clientId, ssl?)` | `(string, string, boolean?) => Promise<void>` | Creates KafkaJS producer and connects |
| `publish(topic, key, value, headers?)` | `(string, string, unknown, Record<string,string>?) => Promise<void>` | Publishes a single message; serializes value to JSON string |
| `publishDLQ(dlqTopic, payload)` | `(string, unknown) => Promise<void>` | Routes a failed payload to the DLQ topic |
| `disconnect()` | `() => Promise<void>` | Disconnects the KafkaJS producer |

---

### KafkaConsumerService

Registered as `etl-mk:kafka-consumer`.

| Method | Signature | Description |
|---|---|---|
| `connect(brokers, groupId, clientId, ssl?)` | `(string, string, string, boolean?) => Promise<void>` | Creates KafkaJS consumer and connects |
| `subscribe(topic)` | `(string) => Promise<void>` | Subscribes to topic from latest offset |
| `run(handler)` | `(handler) => Promise<void>` | Starts consumption with `autoCommit: false` |
| `commit(topic, partition, offset)` | `(string, number, string) => Promise<void>` | Manual offset commit after successful processing |
| `disconnect()` | `() => Promise<void>` | Disconnects the KafkaJS consumer |

---

### MongoWriterService

Registered as `etl-mk:mongo-writer`.

| Method | Signature | Description |
|---|---|---|
| `connect(uri)` | `(string) => Promise<void>` | Opens a MongoClient connection |
| `getDb(dbName)` | `(string) => Db` | Returns a `Db` handle |
| `write(dbName, collectionName, document, writeMode)` | `(string, string, unknown, 'insert'\|'upsert') => Promise<void>` | `insertOne` or `updateOne` with `upsert:true` (by `_id`) |
| `writeDLQ(dbName, dlqCollection, payload)` | `(string, string, unknown) => Promise<void>` | Inserts failed payload to DLQ collection |
| `disconnect()` | `() => Promise<void>` | Closes the MongoClient |

---

### DelegateLoaderService

Registered as `etl-mk:delegate-loader` (transient, used internally).

| Method | Signature | Description |
|---|---|---|
| `loadFromFile(filePath, delegateType?)` | `(string, 'esm'\|'cjs'?) => Promise<IKafkaDelegate>` | Imports ESM (`import()`) or CJS (`require()`) delegate |
| `dispatch(delegate, event, tools, operationType?)` | `(object, unknown, object, string?) => Promise<unknown>` | Routes event to the matching handler export |
| `detectType(filePath)` | `(string) => 'esm'\|'cjs'` | Auto-detects by extension (`.mjs` → esm, `.cjs` → cjs) |

---

### EtlCLIController

Registered as `etl:controller:cli`. Handles `etl:*` CLI actions.

| Method | Description |
|---|---|
| `fill(args)` | Merges CLI flags and env vars into `IArgs` |
| `start(args?)` | Builds `IEtlOptions` from env/flags, calls `EtlPipelineService.start()` |
| `validate(args?)` | Checks required env vars for each active direction; throws on missing |
| `help()` | Renders the full CLI help text from `src/docs/etl-mk.txt` |

---

## CLI commands

### `etl-mk:start` — start the pipeline

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start

# With explicit .env file
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:start --envFile=.env.production

# Development (ts-node)
npx ts-node node_modules/@kozen/engine/dist/bin/kozen.js --action=etl-mk:start
```

### `etl-mk:validate` — validate configuration

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:validate
# Exit code 0 = valid; non-zero = missing required variables (logged to stdout)
```

### `etl-mk:help` — show CLI help

```bash
npx kozen --moduleLoad=@kozen/etl-mk --action=etl-mk:help
```

---

## IoC dependency map

Services registered by `EtlModule.register()`:

```
etl-mk:pipeline           ← EtlPipelineService
  ├─ etl-mk:mongo-to-kafka ← MongoToKafkaService
  │    └─ etl-mk:kafka-producer ← KafkaProducerService
  └─ etl-mk:kafka-to-mongo ← KafkaToMongoService
       ├─ etl-mk:kafka-consumer ← KafkaConsumerService
       ├─ etl-mk:kafka-producer ← KafkaProducerService (shared)
       └─ etl-mk:mongo-writer   ← MongoWriterService

CLI:
  etl:controller:cli      ← EtlCLIController
    ├─ etl-mk:pipeline
    └─ core:file (from @kozen/engine)
```

All registrations are **transient** — a new instance is created per resolution.

---

## Programmatic embedding

### Pattern 1: Embedded in a Kozen app

```typescript
import { IoC } from '@kozen/engine';
import { EtlModule, EtlPipelineService, IEtlOptions } from '@kozen/etl-mk';

const container = new IoC();
const mod = new EtlModule();
const deps = await mod.register(null); // SDK mode: no CLI controller
await container.register(deps!);

const pipeline = await container.resolve<EtlPipelineService>('etl-mk:pipeline');

const options: IEtlOptions = {
  mk: {
    source:      { uri: process.env.MONGO_URI!, database: 'mydb', collection: 'orders' },
    destination: { brokers: process.env.KAFKA_BROKERS!, topic: 'orders.events' },
    delegate:    './delegates/orders.mjs'
  }
};

await pipeline.start(options); // long-running; resolves when stream ends
```

### Pattern 2: Combined with @kozen/secret in delegates

```javascript
// delegates/encrypted-archive.mjs
// Module loaded with: KOZEN_MODULE_LOAD=@kozen/secret,@kozen/etl-mk

export async function message(msg, tools) {
  const secretMgr = await tools.assistant?.resolve('secret:manager');
  const encKey = await secretMgr?.resolve('ARCHIVE_ENCRYPTION_KEY');

  return {
    payload:  encrypt(JSON.stringify(msg), encKey),
    archivedAt: new Date()
  };
}
```

Start command:
```bash
KOZEN_MODULE_LOAD=@kozen/secret,@kozen/etl-mk \
  npx kozen --action=etl-mk:start
```

### Architecture flow

```
MongoDB change stream
  ↓
MongoToKafkaService.onChange()
  ↓ delegate.insert / update / delete / on
  ↓ return payload (null → skip)
KafkaProducerService.publish(topic, key, payload, headers)
  ↓ error → publishDLQ(dlqTopic, error envelope)

─────────────────────────────────────────────────────

KafkaConsumerService.run(handler)
  ↓ each message (autoCommit=false)
KafkaToMongoService.onMessage() — retry loop
  ↓ delegate.message / on / default
  ↓ return document (null → skip)
MongoWriterService.write(db, collection, document, writeMode)
  ↓ success → KafkaConsumerService.commit(topic, partition, offset)
  ↓ max retries → publishDLQ() + commit (at-least-once guaranteed)
```
