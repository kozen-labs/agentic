---
name: "@kozen/trigger — Delegates"
description: >
  Complete guide to writing Kozen trigger delegate files: event handler function names
  (insert, update, replace, delete, invalidate, default/on), the ITriggerTools parameter
  (assistant, db, collection, dbName, collectionName, flow, changeStream), ESM and CJS
  delegate formats, handler dispatch rules, accessing MongoDB directly from handlers,
  resolving Kozen services via tools.assistant, and programmatic use of ChangeStreamService.
category: kozen-trigger
tags:
  - kozen
  - trigger
  - delegate
  - change-stream
  - event-handler
  - ITriggerTools
  - ESM
  - CJS
---

# @kozen/trigger — Delegates

A delegate is the business-logic file that Kozen binds to a MongoDB Change Stream. It is a
plain JavaScript or TypeScript module where each exported function name maps to a MongoDB
`operationType`. Kozen calls the matching function automatically and injects a `tools` object
giving the handler access to logging, the IoC container, and MongoDB connection handles.

---

## Event handler names

| Export name | MongoDB operationType | Notes |
|---|---|---|
| `insert` | `insert` | New document inserted |
| `update` | `update` | Document fields updated |
| `replace` | `replace` | Entire document replaced |
| `delete` | `delete` | Document deleted |
| `invalidate` | `invalidate` | Change stream invalidated (collection dropped, renamed) |
| `default` or `on` | (any) | Runs after the operation-specific handler as a catch-all |

Handler dispatch rules:
1. Kozen calls the operation-specific export if it exists.
2. Kozen then calls `default` or `on` (whichever is exported) for shared logic.
3. Missing handlers are skipped without error.
4. Runtime errors inside a handler are caught; the stream keeps running.

You can also add semantic aliases inside your delegate (e.g., name a function `create` that
internally calls your `insert` logic) to communicate intent to teammates.

---

## ITriggerTools — the second parameter

Every handler receives `(change, tools)` where `tools` is:

```typescript
interface ITriggerTools {
  assistant?: IIoC;              // Kozen IoC container — resolve any loaded module service
  flow?: string;                 // Unique correlation ID for this change event
  changeStream?: ChangeStream;   // The underlying MongoDB Change Stream (rarely needed)
  collectionName?: string;       // Target collection name
  collection?: Collection;       // Ready-to-use MongoDB Collection handle
  dbName?: string;               // Target database name
  db?: Db;                       // Ready-to-use MongoDB Db handle
}
```

`tools.db` and `tools.collection` are connected MongoDB handles — use them directly for
reads or writes without managing connections:

```javascript
export async function insert(change, tools) {
  const { db, collection, dbName, collectionName, flow, assistant } = tools;

  // Direct MongoDB write using the ready-made handle
  await db.collection('audit').insertOne({
    event: 'insert',
    source: `${dbName}.${collectionName}`,
    documentKey: change.documentKey,
    timestamp: new Date()
  });

  assistant?.logger?.info({
    flow,
    message: 'Audit record written',
    data: { documentKey: change.documentKey }
  });
}
```

`tools.assistant` is the full Kozen IoC container. Use it to resolve any module service that
was loaded alongside the trigger:

```javascript
export async function insert(change, tools) {
  // Resolve a secret (requires @kozen/secret loaded via KOZEN_MODULE_LOAD)
  const secretManager = await tools.assistant?.resolve('secret:manager');
  const apiKey = await secretManager?.resolve('MY_API_KEY');

  // Call an external service using the resolved key
  await fetch('https://api.example.com/notify', {
    method: 'POST',
    headers: { Authorization: `Bearer ${apiKey}` },
    body: JSON.stringify({ event: change })
  });
}
```

---

## ESM delegate format

Use for `.mjs` files or projects with `"type": "module"` in their `package.json`:

```javascript
// mydelegate.mjs

export async function insert(change, tools) {
  const { assistant, dbName, collectionName, flow } = tools;

  assistant?.logger?.info({
    flow,
    message: 'Document inserted',
    data: {
      database: dbName,
      collection: collectionName,
      document: change.fullDocument
    }
  });
}

export async function update(change, tools) {
  assistant?.logger?.debug({
    flow: tools.flow,
    message: 'Document updated',
    data: { updatedFields: change.updateDescription?.updatedFields }
  });
}

export async function delete_(change, tools) {  // 'delete' is a reserved word in JS
  // Handle deletion
}

// Catch-all: runs after every specific handler
export default function on(change, tools) {
  tools.assistant?.logger?.debug({
    flow: tools.flow,
    message: `Event handled: ${change.operationType}`
  });
}
```

---

## CJS delegate format

Use for `.cjs` files or legacy CommonJS projects:

```javascript
// mydelegate.cjs

async function insert(change, tools) {
  const { assistant, dbName, collectionName, flow } = tools;

  assistant?.logger?.info({
    flow,
    message: 'Document inserted',
    data: { database: dbName, collection: collectionName }
  });
}

async function update(change, tools) {
  // Handle updates
}

function handleAny(change, tools) {
  tools.assistant?.logger?.debug({
    flow: tools.flow,
    message: `Handled ${change.operationType}`
  });
}

module.exports = { insert, update, on: handleAny };
```

---

## Module system detection

Kozen determines the delegate module system using this priority order:

1. `KOZEN_TRIGGER_DELEGATE_TYPE` env var or `--delegateType` CLI flag (explicit override)
2. File extension: `.mjs` → ESM, `.cjs` → CJS
3. Nearest `package.json` `type` field: `"module"` → ESM, `"commonjs"` or missing → CJS

Accepted values for the override: `esm`, `mjs`, `module` (all treated as ESM);
`commonjs`, `cjs` (treated as CJS).

When placing the delegate alongside application code, the simplest approach is a minimal
`package.json` that declares the module type:

```json
{ "type": "module" }
```

---

## Change event structure

The `change` parameter is the raw MongoDB Change Stream document:

```typescript
// Common fields across all event types
change.operationType        // 'insert' | 'update' | 'replace' | 'delete' | 'invalidate'
change.documentKey._id      // ObjectId of the affected document
change.ns.db                // Database name
change.ns.coll              // Collection name
change.clusterTime          // MongoDB server timestamp

// insert / replace
change.fullDocument         // The complete document after the operation

// update
change.updateDescription.updatedFields    // Map of changed field paths → new values
change.updateDescription.removedFields    // Array of removed field paths
change.updateDescription.truncatedArrays  // Array resize information

// replace
change.fullDocumentBeforeChange           // Previous document (if pre/post image enabled)
```

---

## Programmatic API — ChangeStreamService

Use `ChangeStreamService` directly without the Kozen CLI when integrating into an existing
Node.js application:

```typescript
import { ChangeStreamService, ITriggerOptions } from '@kozen/trigger';

const service = new ChangeStreamService();

const options: ITriggerOptions = {
  uri:            process.env.KOZEN_TRIGGER_URI!,
  database:       'mydb',
  collection:     'orders',
  file:           '/path/to/mydelegate.mjs',
  delegateType:   'esm',       // optional; auto-detected from file extension
};

await service.start(options);
```

The service manages the Change Stream connection internally. To integrate with an existing
IoC container or logger, extend the service or construct it with the Kozen `KzModule` pattern.
