---
name: "@kozen/trigger — Configuration"
description: >
  Complete configuration reference for @kozen/trigger: all KOZEN_TRIGGER_* environment
  variables (FILE, DATABASE, COLLECTION, URI, KEY, DELEGATE_TYPE), .env file patterns,
  ITriggerOptions interface, KOZEN_LOG_* variables, deployment best practices (PM2,
  systemd, Docker), and security guidelines for least-privilege MongoDB connections.
category: kozen-trigger
tags:
  - kozen
  - trigger
  - environment-variables
  - ITriggerOptions
  - configuration
  - PM2
  - deployment
---

# @kozen/trigger — Configuration

## Environment variables

### Required variables

| Variable | CLI flag | Description |
|---|---|---|
| `KOZEN_TRIGGER_FILE` | `--file` | Absolute path to the delegate JavaScript file |
| `KOZEN_TRIGGER_DATABASE` | `--database` | MongoDB database to watch |
| `KOZEN_TRIGGER_COLLECTION` | `--collection` | MongoDB collection to watch |
| `KOZEN_TRIGGER_URI` | `--uri` | MongoDB connection string |

### Optional variables

| Variable | CLI flag | Default | Description |
|---|---|---|---|
| `KOZEN_TRIGGER_KEY` | `--key` | `trigger:delegate:default` | IoC key for the delegate service |
| `KOZEN_TRIGGER_DELEGATE_TYPE` | `--delegateType` | auto-detected | Force ESM or CJS loading: `esm`, `mjs`, `module`, `commonjs`, `cjs` |

### Logger variables (from engine)

| Variable | Default | Description |
|---|---|---|
| `KOZEN_LOG_LEVEL` | `DEBUG` | Console log verbosity |
| `KOZEN_LOG_TYPE` | `object` | Output format: `object` or `json` |
| `KOZEN_LOG_MDB_ENABLED` | `true` | Persist logs to MongoDB (set `false` when `MDB_URI` is unavailable) |

---

## .env file example

```bash
# Module
KOZEN_MODULE_LOAD=@kozen/trigger

# Required trigger configuration
KOZEN_TRIGGER_DATABASE=production
KOZEN_TRIGGER_COLLECTION=orders
KOZEN_TRIGGER_URI=mongodb+srv://trigger-user:pass@cluster.mongodb.net/?retryWrites=true&w=majority
KOZEN_TRIGGER_FILE=/opt/myapp/delegates/orders.mjs

# Optional
KOZEN_TRIGGER_DELEGATE_TYPE=esm
KOZEN_LOG_LEVEL=INFO
KOZEN_LOG_TYPE=object
KOZEN_LOG_MDB_ENABLED=false
```

Load the file:

```bash
npx kozen --action=trigger:start --envFile=/opt/myapp/.env.trigger
```

---

## ITriggerOptions interface

```typescript
export interface ITriggerOptions {
  uri:            string;   // MongoDB connection string
  database:       string;   // Database name
  collection:     string;   // Collection name
  file:           string;   // Absolute path to the delegate file
  delegateType?:  string;   // 'esm' | 'cjs' — overrides auto-detection
  key?:           string;   // IoC key for the delegate (default: 'trigger:delegate:default')
}
```

---

## CLI command

```bash
# Start using .env (recommended)
npx kozen --moduleLoad=@kozen/trigger --action=trigger:start --envFile=.env

# Start with inline arguments
npx kozen --moduleLoad=@kozen/trigger --action=trigger:start \
  --uri=mongodb+srv://user:pass@cluster.mongodb.net/ \
  --database=mydb \
  --collection=events \
  --file=/absolute/path/to/delegate.mjs

# Show help
npx kozen --moduleLoad=@kozen/trigger --action=trigger:help
```

---

## Deployment patterns

### PM2 (recommended for long-running processes)

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'kozen-trigger',
    script: 'node_modules/@kozen/engine/dist/bin/kozen.js',
    args: '--action=trigger:start',
    env: {
      KOZEN_MODULE_LOAD: '@kozen/trigger',
      KOZEN_TRIGGER_URI: process.env.MONGO_URI,
      KOZEN_TRIGGER_DATABASE: 'production',
      KOZEN_TRIGGER_COLLECTION: 'orders',
      KOZEN_TRIGGER_FILE: '/opt/app/delegates/orders.mjs',
      KOZEN_LOG_LEVEL: 'INFO'
    },
    restart_delay: 5000,
    max_restarts: 10
  }]
};
```

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

### systemd

```ini
# /etc/systemd/system/kozen-trigger.service
[Unit]
Description=Kozen MongoDB Trigger
After=network.target

[Service]
Type=simple
User=app
WorkingDirectory=/opt/app
EnvironmentFile=/opt/app/.env.trigger
ExecStart=/usr/bin/npx kozen --action=trigger:start
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable kozen-trigger
systemctl start kozen-trigger
```

### Docker

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY delegates/ ./delegates/
CMD ["npx", "kozen", "--action=trigger:start"]
```

```bash
docker run -d \
  -e KOZEN_MODULE_LOAD=@kozen/trigger \
  -e KOZEN_TRIGGER_URI="mongodb+srv://..." \
  -e KOZEN_TRIGGER_DATABASE=production \
  -e KOZEN_TRIGGER_COLLECTION=orders \
  -e KOZEN_TRIGGER_FILE=/app/delegates/orders.mjs \
  -v /host/delegates:/app/delegates \
  my-kozen-trigger-image
```

---

## Security best practices

1. **Use a dedicated MongoDB user for triggers.** Grant only the minimum permissions:
   `read` on the watched collection, and write only on collections the delegate updates.

2. **Never embed credentials in the delegate file.** Use `tools.assistant?.resolve()` to
   fetch secrets from `@kozen/secret` instead.

3. **Keep `KOZEN_LOG_LEVEL=INFO` or `WARN` in production.** Avoid logging full document
   payloads at DEBUG level in production systems — change events can contain PII.

4. **Use TLS in `KOZEN_TRIGGER_URI`.** Ensure the connection string includes
   `tls=true` or uses `mongodb+srv://` for Atlas connections.

5. **Run as a non-root user.** Use a dedicated OS user with minimal permissions for the
   process, especially when reading `KOZEN_TRIGGER_FILE` from disk.
