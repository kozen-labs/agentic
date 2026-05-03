---
name: Kozen Engine — CLI and MCP Applications
description: >
  Complete guide to running Kozen through its two primary interfaces: the CLI application
  (--action routing, --moduleLoad, --type, --envFile, --stack, --config, all core CLI options)
  and the MCP application (--type=mcp, JSON server config for VS Code and Claude Desktop,
  KOZEN_APP_TYPE, module compatibility, stdio transport). Also covers the structured logger
  CLI (--action=logger:log) and programmatic LoggerService API.
category: kozen-engine
tags:
  - kozen
  - CLI
  - MCP
  - model-context-protocol
  - logger
  - npx-kozen
  - module-loading
  - VS-Code-MCP
---

# Kozen Engine — CLI and MCP Applications

## Installation

```bash
# Global install — kozen available from any directory
npm install -g @kozen/engine
kozen --action=help

# Local project install — use npx
npm install @kozen/engine
npx kozen --action=help

# One-off without install
npx -y @kozen/engine --action=help

# Bun
bun add -g @kozen/engine
bunx kozen --action=help

# Deno
deno run --allow-env --allow-read --allow-net npm:@kozen/engine --action=help
```

---

## CLI application

### Invocation syntax

```bash
kozen [--type=cli] --action=<[module:]action> [options]
```

`--type=cli` is the default and can be omitted. The `--action` parameter is the only required
argument.

### Action routing

The `--action` value follows the pattern `module:method`:

- `--action=trigger:start` → resolve the `trigger` module's CLI controller, call its `start` method
- `--action=secret:get` → resolve the `secret` module's CLI controller, call its `get` method
- `--action=logger:log` → resolve the `logger` module's CLI controller, call its `log` method
- `--action=help` → resolve the engine's built-in help controller

When only `--action=method` (no colon) is given, the module alias must be provided separately
via `--module=module-alias` or `KOZEN_MODULE` env var.

### Core CLI options

| Option | Env var | Default | Description |
|---|---|---|---|
| `--action=<module:method>` | `KOZEN_ACTION` | (required) | Action to execute |
| `--moduleLoad=<pkg[,pkg]>` | `KOZEN_MODULE_LOAD` | (none) | Comma-separated npm package names to load |
| `--module=<alias>` | (none) | (none) | Shorthand to set active controller alias |
| `--type=<cli\|mcp>` | `KOZEN_APP_TYPE` | `cli` | Application interface type |
| `--config=<path>` | `KOZEN_CONFIG` | `cfg/config.json` | Path to JSON config file |
| `--stack=<env>` | `KOZEN_STACK` | `dev` | Environment identifier (`dev`, `test`, `prod`) |
| `--project=<id>` | `KOZEN_PROJECT` | auto-generated | Project ID for log correlation |
| `--envFile=<path>` | (none) | `.env` | `.env` file to load |
| `--skipDotEnv=true` | `KOZEN_SKIP_DOT_ENV` | `false` | Skip loading the `.env` file |
| `--flow=<id>` | (none) | auto-generated | Correlation ID override for this invocation |

### Module loading — three methods

**Method 1: CLI flag**

```bash
npx kozen --moduleLoad=@kozen/secret,@kozen/trigger --action=secret:get --key=MY_KEY
```

**Method 2: Environment variable**

```bash
export KOZEN_MODULE_LOAD=@kozen/secret,@kozen/trigger
npx kozen --action=secret:get --key=MY_KEY
```

**Method 3: .env file**

```bash
# .env
KOZEN_MODULE_LOAD=@kozen/secret,@kozen/trigger
```

```bash
npx kozen --action=secret:get --key=MY_KEY
# kozen auto-reads .env in the current directory
```

Modules not loaded at startup are unavailable. Attempting to call an unloaded module's action
produces a "controller not found" error.

### CLI header banner

Every CLI invocation prints a diagnostic banner showing engine and loaded module versions:

```
#    ##....##..#######..########.########.##....##
#    ...
#    Engine: Task Execution Framework
#    Version: 1.1.15
#    License: MIT
#    Homepage: https://github.com/kozen-labs/engine/wiki
...............................................................................
#    Module: 'secret' from '@kozen/secret' package
#    Version: 1.0.4
#    Wiki: https://github.com/mongodb-industry-solutions/kozen-secret/wiki
```

Use this banner to verify version compatibility before filing bug reports.

### Practical CLI examples

```bash
# Get help from a module
npx kozen --moduleLoad=@kozen/trigger --action=trigger:help

# Start a MongoDB trigger (reads .env automatically)
npx kozen --moduleLoad=@kozen/trigger --action=trigger:start

# Retrieve a secret using SCRAM
npx kozen --moduleLoad=@kozen/secret --action=secret:get \
  --key=MY_SECRET --driver=mdb

# Run IAM rectification in CI (env vars override .env)
export KOZEN_MODULE_LOAD=@kozen/iam-rectification
export KOZEN_STACK=prod
kozen --action=iam-rectification:verify --uriEnv=MONGO_URI --permissions=read,insert

# Load multiple modules at once
npx kozen \
  --moduleLoad=@kozen/iam-rectification,@kozen/trigger,@kozen/secret \
  --action=iam-rectification:help
```

---

## MCP application

The MCP application exposes Kozen module capabilities as tools for Large Language Models
(LLMs) and Small Language Models (SLMs) via the Model Context Protocol (JSON-RPC over stdio).

### Starting the MCP server

```bash
# CLI flag
npx kozen --type=mcp --moduleLoad=@kozen/secret,@kozen/trigger

# Environment variable (preferred for MCP host configs)
export KOZEN_APP_TYPE=mcp
export KOZEN_MODULE_LOAD=@kozen/secret,@kozen/trigger
npx kozen

# Development with MCP Inspector UI (localhost:5173)
npm run mcp:dev   # → npx -y @modelcontextprotocol/inspector npx ts-node bin/kozen.ts --type=mcp
```

### MCP server configuration for VS Code / Claude Desktop

Use this JSON format in your editor's MCP servers config file:

**Option A: ts-node (development, no build required)**

```json
{
  "servers": {
    "kozen": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "ts-node", "bin/kozen.ts", "--type=mcp"],
      "env": {
        "MDB_URI": "mongodb+srv://<user>:<pass>@<cluster>/<db>?retryWrites=true&w=majority",
        "KOZEN_LOG_LEVEL": "NONE",
        "KOZEN_MODULE_LOAD": "@kozen/secret,@kozen/trigger,@kozen/iam-rectification"
      }
    }
  }
}
```

**Option B: pre-installed global package (production)**

```json
{
  "servers": {
    "kozen": {
      "type": "stdio",
      "command": "npx",
      "args": ["kozen"],
      "env": {
        "MDB_URI": "mongodb+srv://<user>:<pass>@<cluster>/<db>?retryWrites=true&w=majority",
        "MDB_MASTER_KEY": "<base64-master-key>",
        "KOZEN_APP_TYPE": "mcp",
        "KOZEN_LOG_LEVEL": "NONE",
        "KOZEN_MODULE_LOAD": "@kozen/secret,@kozen/trigger,@kozen/iam-rectification"
      }
    }
  }
}
```

### MCP-specific environment variables

| Variable | Description |
|---|---|
| `KOZEN_APP_TYPE=mcp` | Activates MCP application type (same as `--type=mcp`) |
| `KOZEN_LOG_LEVEL=NONE` | Suppress all console output (required for MCP — stdout is the protocol channel) |
| `KOZEN_MODULE_LOAD` | Comma-separated modules to expose as MCP tools |

Setting `KOZEN_LOG_LEVEL=NONE` is essential for MCP. Any text written to stdout by the
logger will corrupt the JSON-RPC stream and break the MCP connection.

### Module MCP compatibility

Not all modules support MCP. A module exposes MCP tools only if it:
1. Implements a `MyMCPController` that extends `KzController`
2. Registers `MyMCPController` in its `mcp.json` dependency config
3. Merges `mcp.json` into the dependency map when `config.type === 'mcp'`

| Module | MCP support | MCP tools |
|---|---|---|
| `@kozen/secret` | Yes | `kozen_secret_select`, `kozen_secret_save` |
| `@kozen/trigger` | No | (CLI only) |
| `@kozen/iam-rectification` | Yes | `kozen_iam_rectify_scram`, `kozen_iam_rectify_x509` |

---

## Logger

### CLI usage

```bash
# Simple info message
kozen --action=logger:log --level=info --message="Application started"

# Debug with flow correlation
kozen --action=logger:log --level=debug --flow=K2025080509533 \
  --message="User login" --data='{"userId":"123"}'

# Error from a JSON file
kozen --action=logger:log --level=error --file=./logs/error-report.json

# Categorized security warning
kozen --action=logger:log --category=SECURITY --level=warn \
  --message="Failed login attempt"
```

Logger CLI options:

| Option | Env var | Default | Description |
|---|---|---|---|
| `--level` | `KOZEN_LOG_LEVEL` | `debug` | `ERROR\|WARN\|INFO\|DEBUG\|VERBOSE` |
| `--message` | (none) | (none) | Text message |
| `--data` | (none) | (none) | JSON payload (structured data) |
| `--file` | `KOZEN_LOG_FILE` | (none) | Path to JSON file to ingest as log entry |
| `--flow` | (none) | auto-generated | Correlation flow ID |
| `--category` | (none) | `CLI.LOGGER` | Tag for filtering |
| `--src` | (none) | (none) | Source location identifier |

### Programmatic LoggerService API

```typescript
import { LoggerService, ILogLevel } from '@kozen/engine';

// Default setup — reads env vars for configuration
const logger = new LoggerService();
logger.info('Application started');
logger.warn({ message: 'Low disk space', data: { available: '2GB' } });
logger.error({ src: 'MyService:method', message: 'Operation failed', data: { error } });

// Custom dual-output setup
const logger = new LoggerService({
  level: ILogLevel.DEBUG,
  category: 'PIPELINE',
  console: { enabled: true, level: ILogLevel.DEBUG },
  mdb: {
    enabled: true,
    uri: 'MDB_URI',              // Resolves to process.env.MDB_URI
    database: 'kozen',
    collection: 'pipeline_logs',
    level: ILogLevel.DEBUG
  }
});

// Runtime reconfiguration
logger.configure({ level: ILogLevel.WARN, category: 'PIPELINE' });

// Flush pending writes before exit
await Promise.all(logger.stack);
```

Log entry structure:

```typescript
interface ILogInput {
  flow?: string;       // Correlation ID
  src?: string;        // 'Module:Class:method'
  message: string;     // Human-readable description
  category?: string;   // VCategory.* constant
  data?: any;          // Structured payload
}
```

### Log output destinations

| Destination | Processor class | Config key | Notes |
|---|---|---|---|
| Console (stdout/stderr) | `ConsoleLogProcessor` | `console` | Default always on |
| MongoDB | `MongoDBLogProcessor` | `mdb` | Requires `MDB_URI` env var |
| File | `FileLogProcessor` | `file` | Used for report ingestion |
| Hybrid fan-out | `HybridLogProcessor` | (internal) | Routes to all enabled processors |

Processors are independent: failure in MongoDB logging does not affect console output.

### Best practices for logging

- Always set `KOZEN_LOG_LEVEL=NONE` when running as MCP server (stdout is the protocol channel).
- Use `KOZEN_LOG_TYPE=json` in production for log aggregator compatibility.
- Use consistent `category` values (`APP`, `SECURITY`, `PIPELINE`) across all modules for filtering.
- Include `flow` in every multi-step operation log; use `this.getId()` once per invocation.
- Set separate levels for `console.level` and `mdb.level` to reduce MongoDB write volume in
  high-frequency operations.
- Always call `await Promise.all(logger.stack)` before process exit to flush async writes.
