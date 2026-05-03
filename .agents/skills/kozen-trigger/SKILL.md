---
name: kozen-trigger
description: >
  Reference skill for @kozen/trigger — a Kozen module that runs self-hosted MongoDB
  Change Stream triggers on your own infrastructure (alternative to Atlas Triggers). Covers
  the delegate pattern (event handler functions: insert, update, replace, delete, default/on),
  ITriggerTools (assistant, db, collection, flow), ITriggerOptions configuration, ESM vs CJS
  delegate loading, CLI command (trigger:start), all environment variables, programmatic API
  (ChangeStreamService), and how to compose the module with other Kozen modules.
created: 2026-05-03
updated: 2026-05-03
---

# @kozen/trigger — Self-Hosted MongoDB Triggers

`@kozen/trigger` enables MongoDB Change Stream-based triggers that run on your own
infrastructure. Each trigger is a JavaScript/TypeScript delegate file where exported
function names map to MongoDB operation types. Kozen manages the Change Stream lifecycle,
error recovery, and dependency injection into every handler call.

---

## Routing table

| Signal | Reference |
|---|---|
| delegate, event handler, insert handler, update handler, delete handler, replace handler, default handler, on handler, ITriggerTools, assistant tools, flow tools, ITriggerDelegate, change stream event, operationType, trigger:start, ChangeStreamService, programmatic trigger, library trigger, trigger as dependency | `references/delegates.md` |
| KOZEN_TRIGGER_FILE, KOZEN_TRIGGER_DATABASE, KOZEN_TRIGGER_COLLECTION, KOZEN_TRIGGER_URI, KOZEN_TRIGGER_KEY, ESM CJS delegate, delegateType, KOZEN_TRIGGER_DELEGATE_TYPE, .env trigger, trigger environment variables, trigger configuration, ITriggerOptions | `references/configuration.md` |

---

## When to use this skill

- Writing a delegate file to react to MongoDB change events
- Starting a Kozen trigger from the CLI
- Using `ChangeStreamService` directly in a Node.js project
- Configuring the module's environment variables in a `.env` file or CI pipeline
- Understanding how ESM vs CJS delegate loading works

---

## Quick start

```bash
npm install @kozen/trigger
```

```bash
# Start a trigger (reads KOZEN_TRIGGER_* from .env)
npx kozen --moduleLoad=@kozen/trigger --action=trigger:start --envFile=.env
```

Minimal `.env`:

```bash
KOZEN_MODULE_LOAD=@kozen/trigger
KOZEN_TRIGGER_DATABASE=mydb
KOZEN_TRIGGER_COLLECTION=orders
KOZEN_TRIGGER_URI=mongodb+srv://user:pass@cluster.mongodb.net/
KOZEN_TRIGGER_FILE=/absolute/path/to/mydelegate.mjs
KOZEN_LOG_LEVEL=INFO
```

Minimal ESM delegate (`mydelegate.mjs`):

```javascript
export async function insert(change, tools) {
  tools.assistant?.logger?.info({
    flow: tools.flow,
    message: 'New order received',
    data: { doc: change.fullDocument }
  });
}
```

---

## Do & Don't

**Do:**
- Keep delegate handlers small and idempotent — change streams can replay events.
- Use `tools.db` or `tools.collection` for direct MongoDB access inside handlers.
- Include `flow: tools.flow` in every log call for end-to-end tracing.
- Run the process under PM2, systemd, or a container orchestrator for resilience.
- Use least-privilege credentials: read-only if handlers do not write back.

**Don't:**
- Never perform heavy synchronous I/O inside a handler; offload to queues or background tasks.
- Never rely on in-memory state between handler invocations (the process may restart).
- Never use relative file paths for `KOZEN_TRIGGER_FILE` — always absolute paths.
