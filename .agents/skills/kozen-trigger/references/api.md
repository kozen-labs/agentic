---
name: "@kozen/trigger — Public API"
description: >
  Complete API reference for @kozen/trigger: all exported classes and interfaces with full
  constructor signatures and method documentation, ChangeStreamService (start, onChange, stop),
  ITriggerDelegate (all 18 event handler names and dispatch rules), ITriggerOptions (flow, opt,
  mdb), ITriggerTools (assistant, db, collection, dbName, collectionName, flow, changeStream),
  TriggerModule, CLI command with all flags, programmatic use patterns, embedding in other
  Node.js projects, and composing with other Kozen modules.
category: kozen-trigger
tags:
  - kozen
  - trigger
  - ChangeStreamService
  - ITriggerDelegate
  - ITriggerOptions
  - ITriggerTools
  - change-stream
  - MongoDB
  - CLI
  - programmatic
---

# @kozen/trigger — Public API

## All public exports

```typescript
import {
  TriggerModule,        // KzModule subclass — the Kozen module entry point
  ChangeStreamService,  // Service: manages the MongoDB Change Stream lifecycle
  ITriggerDelegate,     // Interface: the event handler map a delegate file must export
  ITriggerOptions,      // Interface: options passed to ChangeStreamService.start()
  ITriggerTools,        // Interface: context object injected into every handler
} from '@kozen/trigger';
```

---

## TriggerModule

The Kozen module entry point.

**Constructor:**
```typescript
constructor(dependency?: any)
```
Reads `package.json` to populate `metadata` (name, version, summary, author, license, uri).
Sets `metadata.alias = 'trigger'` for action routing (`--action=trigger:start`).

**`register(config, opts?)`**
```typescript
register(config: IConfig | null, opts?: any): Promise<Record<string, IDependency> | null>
```
Returns the dependency map for the runtime type:
- `config.type === 'cli'` → `{ ...ioc, ...cli }` (ChangeStreamService + CLI controller)
- default → `ioc` only (ChangeStreamService only — no controllers, for library use)

**Note:** `@kozen/trigger` does not include an MCP controller. The module does not expose
an MCP interface — it runs as a long-lived CLI process, not as a request-response server.

---

## ChangeStreamService

The core service. Manages a MongoDB Change Stream connection: connects, listens for events,
dispatches to the delegate, and handles cleanup.

**Constructor:**
```typescript
constructor(dependency?: {
  assistant: IIoC;
  logger: ILogger;
})
```
When constructed through the IoC container (normal Kozen usage), `assistant` and `logger`
are injected automatically. For standalone use, pass `undefined` — internal operations use
`console` as fallback.

**Properties (internal, not typically accessed directly):**

| Property | Type | Description |
|---|---|---|
| `client` | `MongoClient \| undefined` | Active MongoDB connection (created by `start()`) |
| `changeStream` | `ChangeStream \| undefined` | Live change stream cursor (created by `start()`) |
| `options` | `ITriggerOptions \| undefined` | Options passed to the last `start()` call |
| `assistant` | `IIoC \| null` | IoC container (injected) |
| `logger` | `ILogger \| null` | Structured logger (injected) |

**Methods:**

### `start(options?)`
```typescript
start(options?: ITriggerOptions): Promise<void>
```

Opens the MongoDB connection, resolves the delegate from the IoC container or from a file
path, opens a Change Stream on the configured collection, and starts listening.

**What `start()` does internally:**
1. Reads MongoDB connection options from `options.mdb` (uri, database, collection).
2. Creates and connects a `MongoClient`.
3. Resolves the delegate: if `options.opt` is an `IDependency` descriptor, resolves from
   IoC; if `options.file` is set, loads the file as an ESM/CJS module.
4. Opens `collection.watch()` with `{ fullDocument: 'updateLookup' }`.
5. Registers an `onChange` listener on the stream.
6. Awaits indefinitely (the stream keeps the process alive).

**`options` parameter:** See `ITriggerOptions` below.

### `onChange(change, delegate?, tools?)`
```typescript
onChange(
  change: ChangeStreamDocument<Document>,
  delegate?: ITriggerDelegate,
  tools?: ITriggerTools
): Promise<void>
```

Event router. Called for every change event from the stream. Dispatches to the delegate
handler matching `change.operationType`, then to the catch-all. Not typically called
directly — it is the internal listener registered by `start()`.

**Dispatch order:**
1. `delegate[change.operationType]` — operation-specific handler (e.g., `delegate.insert`)
2. If `operationType === 'delete'` and no `delete` handler exists: tries `delegate.remove`
3. `delegate.default` or `delegate.on` — catch-all (runs even if specific handler ran)
4. If no handler found: logs a warning, does not throw

### `stop()`
```typescript
stop(): Promise<void>
```

Closes the change stream cursor and the MongoDB client. Call this for graceful shutdown in
long-running processes (PM2 restart, SIGTERM handlers, etc.).

```typescript
process.on('SIGTERM', async () => {
  await service.stop();
  process.exit(0);
});
```

---

## ITriggerOptions — start options

```typescript
interface ITriggerOptions {
  /**
   * Flow correlation ID for distributed tracing.
   * Passed to tools.flow in every handler call.
   * Auto-generated if not provided.
   */
  flow?: string;

  /**
   * IoC dependency descriptor for the delegate.
   * Used when the delegate is registered in the IoC container (Kozen integration).
   * Example: { target: 'MyDelegate', type: 'class', path: '../delegates' }
   * Alternative to the --file CLI flag.
   */
  opt?: IDependency;

  /**
   * MongoDB connection and target options.
   */
  mdb?: IMdbClientOpt;
}
```

`IMdbClientOpt` (from `@kozen/engine`):
```typescript
interface IMdbClientOpt {
  uri:          string;   // MongoDB connection string
  database:     string;   // Database name
  collection:   string;   // Collection name to watch
  options?:     MongoClientOptions;  // Driver-level options (TLS, auth, etc.)
}
```

---

## ITriggerDelegate — the event handler contract

The object a delegate file must export. All handlers are optional — export only the events
your use case needs.

```typescript
interface ITriggerDelegate {
  /** Catch-all. Runs for every event, after the operation-specific handler. */
  default?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Alternative catch-all name. Use either 'on' or 'default', not both. */
  on?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** New document inserted. change.fullDocument contains the inserted doc. */
  insert?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Document fields updated. change.updateDescription has field-level diff. */
  update?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Entire document replaced. change.fullDocument is the new version. */
  replace?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Document deleted. change.documentKey has the _id. */
  delete?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Alias for delete (used when 'delete' conflicts with JS reserved words in older runtimes). */
  remove?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection dropped. Stream is about to be invalidated. */
  drop?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection renamed. */
  rename?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Database dropped. */
  dropDatabase?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Change stream invalidated (collection dropped/renamed, database dropped). */
  invalidate?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Index created on the collection. */
  createIndexes?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection created. */
  create?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection schema or validator modified (collMod). */
  modify?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Index dropped from the collection. */
  dropIndexes?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection sharded. */
  shardCollection?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection resharded. */
  reshardCollection?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;

  /** Collection shard key refined. */
  refineCollectionShardKey?: (change: ChangeStreamDocument<Document>, tools?: ITriggerTools) => void | Promise<void>;
}
```

**Handler dispatch rules (in order):**
1. Operation-specific handler matching `change.operationType` is called first.
2. If `operationType === 'delete'` and `delegate.delete` is absent: `delegate.remove` is tried.
3. `delegate.default` or `delegate.on` is called as catch-all (after the specific handler, not instead of it).
4. If no handler matches at all: a warning is logged and the event is silently skipped.

---

## ITriggerTools — handler context object

Every handler receives `(change, tools)`. The `tools` object provides everything a handler
needs to interact with the system:

```typescript
interface ITriggerTools {
  /**
   * The Kozen IoC container.
   * Resolve any loaded module service: secret manager, logger, custom services.
   * null when running standalone (without Kozen module loading).
   */
  assistant?: IIoC;

  /**
   * Unique correlation ID for this change event.
   * Pass to every logger call and to downstream services for distributed tracing.
   */
  flow?: string;

  /**
   * The underlying MongoDB Change Stream cursor.
   * Rarely needed in handlers — use for advanced stream control (pause, resume).
   */
  changeStream?: ChangeStream<Document, ChangeStreamDocument<Document>>;

  /** Name of the collection being watched. */
  collectionName?: string;

  /**
   * Ready-to-use MongoDB Collection handle.
   * Write audit records, update related collections, insert derived documents.
   */
  collection?: Collection<Document>;

  /** Name of the database being watched. */
  dbName?: string;

  /**
   * Ready-to-use MongoDB Db handle.
   * Access other collections in the same database without creating a new connection.
   */
  db?: Db;
}
```

**Common patterns using `tools`:**

```javascript
// Write to another collection using tools.db
export async function insert(change, tools) {
  await tools.db.collection('audit_log').insertOne({
    type: 'insert',
    collection: tools.collectionName,
    documentKey: change.documentKey,
    at: new Date()
  });
}

// Resolve another Kozen module service via tools.assistant
export async function update(change, tools) {
  // Requires @kozen/secret loaded alongside @kozen/trigger
  const secrets = await tools.assistant?.resolve('secret:manager');
  const webhookUrl = await secrets?.resolve('WEBHOOK_URL');
  await fetch(webhookUrl, { method: 'POST', body: JSON.stringify(change) });
}

// Use the flow ID for correlated logging
export async function delete_(change, tools) {
  tools.assistant?.logger?.info({
    flow: tools.flow,
    src: 'MyDelegate:delete',
    message: 'Document deleted',
    data: { id: change.documentKey._id }
  });
}
```

---

## CLI command

### `trigger:start`

Start watching a collection:

```bash
kozen --moduleLoad=@kozen/trigger --action=trigger:start \
  --uri="mongodb+srv://user:pass@cluster.mongodb.net/mydb" \
  --database=mydb \
  --collection=orders \
  --file=/path/to/delegate.mjs

# Or entirely via environment variables
KOZEN_MODULE_LOAD=@kozen/trigger
KOZEN_TRIGGER_URI=mongodb+srv://...
KOZEN_TRIGGER_DATABASE=mydb
KOZEN_TRIGGER_COLLECTION=orders
KOZEN_TRIGGER_FILE=/path/to/delegate.mjs
kozen --action=trigger:start
```

### `trigger:help`

```bash
kozen --moduleLoad=@kozen/trigger --action=trigger:help
```

### All CLI flags

| Flag | Env var | Required | Description |
|---|---|---|---|
| `--uri=<uri>` | `KOZEN_TRIGGER_URI` | Yes | MongoDB connection string |
| `--database=<db>` | `KOZEN_TRIGGER_DATABASE` | Yes | Database name |
| `--collection=<col>` | `KOZEN_TRIGGER_COLLECTION` | Yes | Collection name to watch |
| `--file=<path>` | `KOZEN_TRIGGER_FILE` | Yes* | Path to the delegate file |
| `--key=<token>` | `KOZEN_TRIGGER_KEY` | No | IoC token for delegate (alternative to --file) |
| `--delegateType=<type>` | `KOZEN_TRIGGER_DELEGATE_TYPE` | No | `esm`/`mjs` or `cjs`/`commonjs` |
| `--stack=<env>` | `KOZEN_STACK` | No | Environment: dev/test/prod |
| `--project=<id>` | `KOZEN_PROJECT` | No | Project ID for log correlation |
| `--config=<path>` | `KOZEN_CONFIG` | No | Path to cfg/config.json |

\*Either `--file` or `--key` is required.

---

## Programmatic use patterns

### Pattern 1: Standalone Node.js application

```typescript
import { ChangeStreamService, ITriggerDelegate, ITriggerTools } from '@kozen/trigger';

const delegate: ITriggerDelegate = {
  async insert(change, tools) {
    console.log('Inserted:', change.fullDocument);
  },
  async update(change, tools) {
    console.log('Updated fields:', change.updateDescription?.updatedFields);
  },
  async default(change, tools) {
    console.log('Event:', change.operationType, change.documentKey);
  }
};

const service = new ChangeStreamService();

await service.start({
  opt: { target: delegate, type: 'value' },
  mdb: {
    uri:        process.env.MDB_URI!,
    database:   'mydb',
    collection: 'orders'
  }
});

// Graceful shutdown
process.on('SIGTERM', () => service.stop().then(() => process.exit(0)));
```

### Pattern 2: With Kozen IoC container (full feature set)

```typescript
import { IoC } from '@kozen/engine';
import { TriggerModule, ChangeStreamService, ITriggerOptions } from '@kozen/trigger';

// Set up the container
const container = new IoC();
const mod = new TriggerModule();
const deps = await mod.register(null);  // SDK mode
await container.register(deps!);

// Resolve and start
const svc = await container.resolve<ChangeStreamService>('trigger:service');
await svc.start({
  flow: 'my-flow-001',
  mdb: { uri: process.env.MDB_URI!, database: 'events', collection: 'raw' },
  opt: { target: 'MyDelegateClass', type: 'class', path: './delegates' }
});
```

### Pattern 3: Multi-module composition (trigger + secret)

Load both modules with the CLI to give delegates access to the secret manager:

```bash
KOZEN_MODULE_LOAD=@kozen/trigger,@kozen/secret \
KOZEN_TRIGGER_URI=mongodb+srv://... \
KOZEN_TRIGGER_DATABASE=events \
KOZEN_TRIGGER_COLLECTION=raw \
KOZEN_TRIGGER_FILE=./delegate.mjs \
KOZEN_SM_DRIVER=mdb \
MDB_MASTER_KEY=<base64-key> \
kozen --action=trigger:start
```

The delegate then has access to `tools.assistant.resolve('secret:manager')`.

### Pattern 4: Delegate registered in IoC (class-based)

Instead of a file, register the delegate class in the IoC container and reference it by token:

```typescript
// In cfg/config.json or a custom config
{
  "dependencies": {
    "my:delegate": {
      "target": "OrderProcessingDelegate",
      "type": "class",
      "lifetime": "singleton",
      "path": "./delegates"
    }
  }
}
```

```bash
kozen --action=trigger:start \
  --key=my:delegate \            # IoC token, not a file path
  --database=orders \
  --collection=raw \
  --uri=$MDB_URI
```

This enables the delegate to have its own injected dependencies via the IoC container.

---

## Deployment patterns

### PM2

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name:    'kozen-trigger-orders',
    script:  'npx',
    args:    'kozen --action=trigger:start',
    env: {
      KOZEN_MODULE_LOAD:         '@kozen/trigger',
      KOZEN_TRIGGER_URI:         process.env.MDB_URI,
      KOZEN_TRIGGER_DATABASE:    'mydb',
      KOZEN_TRIGGER_COLLECTION:  'orders',
      KOZEN_TRIGGER_FILE:        '/app/delegates/orders.mjs',
      KOZEN_LOG_LEVEL:           'INFO'
    },
    restart_delay: 5000,
    max_restarts:  10
  }]
};
```

### Docker

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY delegates/ ./delegates/
ENV KOZEN_MODULE_LOAD=@kozen/trigger
ENV KOZEN_LOG_TYPE=json
CMD ["npx", "kozen", "--action=trigger:start"]
```
