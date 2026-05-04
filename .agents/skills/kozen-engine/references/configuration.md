---
name: Kozen Engine — Configuration Reference
description: >
  Complete configuration reference for the Kozen engine and its modules: cfg/config.json
  schema, IDependency registration patterns (class/value/ref/auto), all engine environment
  variables (KOZEN_*), multi-environment config files (dev/test/prod), JSON template schema
  for pipeline execution, logger configuration (console and MongoDB processors), and
  configuration best practices for secrets, service lifetimes, and CI/CD.
category: kozen-engine
tags:
  - kozen
  - config.json
  - environment-variables
  - IoC-registration
  - pipeline-template
  - logger-config
  - multi-environment
---

# Kozen Engine — Configuration Reference

## Configuration layers (priority order)

1. CLI arguments (`--key=value`) — highest priority
2. Environment variables (`KOZEN_*`, `MDB_URI`, etc.)
3. `.env` file loaded via `--envFile=path/to/file`
4. `cfg/config.json` (or the file specified by `--config`)
5. Hardcoded defaults — lowest priority

---

## cfg/config.json — main configuration file

```json
{
  "name": "my-pipeline",
  "version": "1.0.0",
  "engine": ">=1.0.0",
  "description": "Optional description",
  "dependencies": [
    {
      "target": "MyService",
      "type": "class",
      "lifetime": "singleton",
      "path": "../../services",
      "dependencies": [
        { "key": "assistant", "target": "IoC",            "type": "ref" },
        { "key": "logger",    "target": "logger:service", "type": "ref" }
      ]
    }
  ]
}
```

### Top-level schema

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Human-readable pipeline / project name |
| `version` | string | Yes | Config schema version (SemVer) |
| `engine` | string | Yes | Minimum engine version required (SemVer range) |
| `description` | string | No | Free-text description |
| `dependencies` | IDependency[] | Yes | IoC service registrations for this config |
| `modules.load` | string[] | No | Additional modules to auto-load on start |
| `type` | IAppType | No | Override app type (`cli` / `mcp`) |

---

## IDependency — complete field reference

`IDependency` is the central data structure of Kozen's IoC system. Every service registered
in `ioc.json`, `cli.json`, `mcp.json`, or `config.json` is described by one `IDependency`
object. Understanding every field is essential for writing correct registration configs.

### All fields

| Field | Type | Description |
|---|---|---|
| `key` | `string` | IoC token used to identify and resolve this dependency. If omitted, inferred from the map key. Format: `alias:service` or `alias:controller:cli`. |
| `target` | `any` | Class name (for `class`), function name (for `function`/`action`), token string (for `ref`/`alias`), or literal value (for `value`). |
| `type` | `IDependencyType` | Registration strategy. See type reference table below. |
| `as` | `IDependencyType` | Alternative type — used when the primary `type` is ambiguous. Rarely needed. |
| `lifetime` | `IDependencyLifetime` | Instance lifecycle: `'singleton'` \| `'transient'` \| `'scoped'`. Default: `'singleton'`. |
| `path` | `string` | Relative path (from the compiled file calling `fix()`) to the **directory** containing the `target` class or function. Required for `class`/`function`/`auto`. Never include the filename. |
| `file` | `string` | Explicit file path when the class is not the default export or when the path cannot be inferred from `target`. Overrides `path` + `target` combination. |
| `template` | `string` | Path template string with `{path}` and `{target}` placeholders (e.g., `"{path}/{target}"`). Used for dynamic path construction. |
| `args` | `IJSON[] \| any[]` | Positional constructor arguments passed **before** the injected `dependencies` object. Use `null` to skip a position. |
| `dependencies` | `IDependency[] \| IDependencyMap` | Services injected into the constructor. Each entry's `key` must match the corresponding constructor parameter name (Awilix uses name-based injection). |
| `regex` | `string` | Regular expression string. Used with `type: 'auto'` to filter which exports are registered. |
| `moduleType` | `IModuleType` | Module system of the file being imported: `'esm'` \| `'mjs'` \| `'module'` \| `'cjs'` \| `'commonjs'`. Required when importing ESM-only packages from a CJS context. |
| `category` | `string` | Grouping label for metadata and log filtering. Follows the `VCategory` format (e.g., `'cli:tool'`). |
| `raw` | `boolean` | When `true`, registers the `target` value as-is without any factory wrapping. Equivalent to `asValue()` regardless of `type`. |

### IDependencyType — registration strategy reference

| `type` value | What the container does | Required fields | Typical `lifetime` |
|---|---|---|---|
| `'class'` | Calls `new Target(injectedDeps, ...args)`. Target must be an exported class. | `target`, `path` | `singleton` or `transient` |
| `'value'` | Registers the `target` (or a `value` field) directly, no instantiation. Use for config objects, constants. | `target` or `value` | N/A (values have no lifetime) |
| `'function'` / `'method'` | Registers the function itself as an object. The function is not called at registration. | `target`, `path` | `singleton` |
| `'action'` | Calls the function on every resolution (factory pattern). The return value is what consumers receive. | `target`, `path` | `transient` |
| `'ref'` / `'alias'` | Injects another already-registered service by its token. Used **only** inside `dependencies` arrays. | `key` (consumer param), `target` (source token) | Inherits from source |
| `'auto'` | Scans the `path` directory and registers every exported class automatically. Use for bulk component registration. | `path` | `singleton` typical |
| `'instance'` / `'object'` | Registers a pre-built object instance directly. No factory is used. | `target` (the instance) | N/A |

### IDependencyLifetime

| Lifetime | Instances | Container behaviour | Use when |
|---|---|---|---|
| `'singleton'` | One per container | Created on first resolution, reused forever | Stateless services, connection clients, logger, any service without per-call state |
| `'transient'` | One per `resolve()` call | New instance every time | Services with per-call state (open cursors, open sockets, per-request accumulators) |
| `'scoped'` | One per scope | Created once within a scope, new instance per new scope | HTTP request handlers, rarely used in Kozen |

### IDependency examples by type

#### `class` — standard service registration

```json
{
  "secret:manager": {
    "target": "SecretManager",
    "type": "class",
    "lifetime": "singleton",
    "path": "../services",
    "args": [{ "type": "aws" }],
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

`args` passes `{ type: "aws" }` as the first constructor argument. `dependencies` injects
`assistant` and `logger` by matching constructor parameter names. The combined call is:
`new SecretManager({ type: "aws" }, { assistant: iocInstance, logger: loggerInstance })`.

#### `ref` — injecting a previously registered service

```json
{ "key": "srvTrigger", "target": "trigger:service", "type": "ref" }
```

Used exclusively inside a `dependencies` array. Tells Awilix: "inject the service registered
as `trigger:service` into the constructor parameter named `srvTrigger`."

#### `auto` — bulk component registration

```json
{
  "components": {
    "path": "../../components",
    "type": "auto",
    "lifetime": "singleton",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

Awilix scans all `.js`/`.ts` files in `../../components`, imports each, and registers every
exported class. Useful for pipeline component directories where adding a new file should
automatically make it available by class name.

#### `value` — static configuration object

```json
{
  "my:app:config": {
    "key": "my:app:config",
    "target": "my:app:config",
    "type": "value",
    "value": {
      "endpoint": "https://api.example.com",
      "timeout": 5000,
      "retries": 3
    }
  }
}
```

Resolve with: `const cfg = await this.assistant?.resolve<{ endpoint: string }>('my:app:config')`.

#### `moduleType` — ESM package in a CJS module

```json
{
  "my:esm:service": {
    "target": "MyEsmService",
    "type": "class",
    "lifetime": "singleton",
    "path": "../services",
    "moduleType": "esm"
  }
}
```

Required when importing an ESM-only package from a compiled CommonJS module (the default
for Kozen modules compiled with `"module": "CommonJS"` in tsconfig).

#### `template` — dynamic path construction

```json
{
  "my:dynamic:service": {
    "target": "MyService",
    "type": "class",
    "template": "{path}/{target}",
    "path": "../services"
  }
}
```

The template is resolved by `fix()` into an absolute path: `<absolute_dir>/services/MyService`.

---

## IDependency schema

```json
{
  "target": "MyClass",
  "type": "class",
  "lifetime": "singleton",
  "path": "../../services",
  "args": [{ "option1": "value1" }],
  "dependencies": [
    { "key": "assistant", "target": "IoC",            "type": "ref" },
    { "key": "logger",    "target": "logger:service", "type": "ref" }
  ]
}
```

- `target`: Class name as exported (not the file name).
- `path`: Relative path from the compiled file calling `this.fix()` to the directory
  containing the class. Always ends without a slash.
- `args`: Array of constructor arguments. Use `null` to skip a positional argument.
  The dependency-injected arguments in `dependencies` are NOT included in `args`.
- `dependencies[].key`: Must match the corresponding constructor parameter name exactly.

### type: ref — reference another registered service

```json
{
  "key": "assistant",
  "target": "IoC",
  "type": "ref"
}
```

Used exclusively inside a `dependencies` array to inject a previously registered service.

### type: auto — scan a directory and register all exports

```json
{
  "path": "../../components",
  "type": "auto",
  "lifetime": "singleton",
  "args": [{}],
  "dependencies": [
    { "key": "assistant", "target": "IoC",            "type": "ref" },
    { "key": "logger",    "target": "logger:service", "type": "ref" }
  ]
}
```

Scans all `.js`/`.ts` files in the given directory and registers every exported class.
Used for auto-registering pipeline components.

### type: value — register a static value

```json
{
  "key": "my:config:value",
  "target": "my:config:value",
  "type": "value",
  "value": { "endpoint": "https://api.example.com" }
}
```

### Lifetime semantics

| Lifetime | Description | Use when |
|---|---|---|
| `singleton` | One instance per IoC container | Stateless services, shared clients, logger |
| `transient` | New instance on every `resolve()` | Stateful operations, stack managers |
| `scoped` | One per request scope | Rarely used; HTTP request handlers |

---

## Engine environment variables

### Core execution variables

| Variable | CLI equivalent | Default | Description |
|---|---|---|---|
| `KOZEN_CONFIG` | `--config` | `cfg/config.json` | Path to the main config file |
| `KOZEN_ACTION` | `--action` | (none) | Action to execute (`module:method`) |
| `KOZEN_MODULE_LOAD` | `--moduleLoad` | (none) | Comma-separated module packages to load |
| `KOZEN_APP_TYPE` | `--type` | `cli` | Application type: `cli` or `mcp` |
| `KOZEN_STACK` | `--stack` | `dev` | Environment identifier |
| `KOZEN_PROJECT` | `--project` | auto-generated | Project ID for log correlation |
| `KOZEN_SKIP_DOT_ENV` | `--skipDotEnv` | `false` | Skip loading `.env` file when `true` |

### Logger variables

| Variable | Description | Default |
|---|---|---|
| `KOZEN_LOG_LEVEL` | Console processor log level: `NONE\|ERROR\|WARN\|DEBUG\|INFO\|ALL` | `DEBUG` |
| `KOZEN_LOG_LEVEL_REMOTE` | MongoDB processor log level | `DEBUG` |
| `KOZEN_LOG_CONSOLE_ENABLED` | Enable/disable console output: `true\|false` | `true` |
| `KOZEN_LOG_MDB_ENABLED` | Enable/disable MongoDB log persistence: `true\|false` | `true` |
| `KOZEN_LOG_TYPE` | Serialization format: `object\|json` | `object` |
| `KOZEN_LOG_FILE` | Path to a JSON file to ingest as a log entry (logger:log command) | (none) |

### MongoDB variables

| Variable | Description |
|---|---|
| `MDB_URI` | MongoDB connection string (referenced as `"uri": "MDB_URI"` in config) |
| `MDB_MASTER_KEY` | Base64-encoded master key for CSFLE operations |

### AWS variables (used by modules)

| Variable | Description |
|---|---|
| `AWS_REGION` | AWS region |
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `AWS_PROFILE` | AWS credentials profile name |

---

## Logger configuration in config.json

```json
{
  "target": "logger:service",
  "type": "class",
  "lifetime": "singleton",
  "path": "../../services",
  "args": [
    {
      "level": "DEBUG",
      "type": "object",
      "console": {
        "enabled": true,
        "level": "DEBUG",
        "skip": "Stack output: \\."
      },
      "mdb": {
        "enabled": true,
        "uri": "MDB_URI",
        "database": "kozen",
        "collection": "logs",
        "level": "DEBUG"
      }
    }
  ],
  "dependencies": [
    { "key": "assistant", "target": "IoC", "type": "ref" }
  ]
}
```

The `uri` field accepts either a literal connection string or the name of an environment
variable (`"MDB_URI"` → resolved to `process.env.MDB_URI`).

### Log levels

| Level | Number | Logs |
|---|---|---|
| `NONE` | 0 | Nothing |
| `ERROR` | 1 | Errors only |
| `WARN` | 2 | Errors + warnings |
| `DEBUG` | 3 | Errors + warnings + debug |
| `INFO` | 4 | All normal logs |
| `ALL` | -1 | Everything including verbose |

A message is logged when its level number is less than or equal to the processor's configured
level number.

---

## Multi-environment config files

Create environment-specific config files in `cfg/`:

```
cfg/
├── config.json         ← default / production
├── config.dev.json     ← development
└── config.test.json    ← CI / testing
```

Select the file at runtime:

```bash
npx kozen --config=cfg/config.dev.json --action=my-module:execute
# or via env var:
export KOZEN_CONFIG=cfg/config.dev.json
```

Development config example (`cfg/config.dev.json`):

```json
{
  "name": "my-project-dev",
  "version": "1.0.0",
  "engine": ">=1.0.0",
  "dependencies": [
    {
      "target": "logger:service",
      "type": "class",
      "lifetime": "singleton",
      "path": "../../services",
      "args": [{
        "console": { "enabled": true, "level": "ALL" },
        "mdb":     { "enabled": false }
      }],
      "dependencies": [{ "key": "assistant", "target": "IoC", "type": "ref" }]
    }
  ]
}
```

Production config (`cfg/config.json`) typically enables MongoDB logging and sets level to
`WARN` or `INFO` to reduce write volume.

---

## .env file patterns

Use `.env` files for local development secrets. Never commit real credentials.

```bash
# .env (local development)
KOZEN_MODULE_LOAD=@kozen/secret,@kozen/trigger
KOZEN_STACK=dev
KOZEN_LOG_LEVEL=DEBUG
KOZEN_LOG_MDB_ENABLED=false

MDB_URI=mongodb://localhost:27017/kozen-dev
```

Load a non-default .env file:

```bash
npx kozen --action=my-module:execute --envFile=.env.staging
```

---

## Pipeline template schema

Pipeline templates are JSON files placed in `cfg/templates/`. They describe declarative
pipelines that the engine can execute.

```json
{
  "name": "my-pipeline",
  "description": "What this pipeline does",
  "version": "1.0.0",
  "engine": "1.0.0",
  "release": "20240101",
  "requires": [],
  "stack": {
    "orchestrator": "Node",
    "input": [
      { "type": "environment", "name": "REQUIRED_VAR" }
    ],
    "setup": [
      { "type": "environment", "name": "aws:region", "value": "AWS_REGION", "default": "us-east-1" }
    ],
    "components": [
      {
        "name": "MyComponent",
        "input": [
          { "type": "environment", "name": "projectName", "value": "PROJECT_NAME" },
          { "type": "value",       "name": "message",     "value": "Hello" },
          { "type": "secret",      "name": "apiKey",      "value": "MY_API_KEY" },
          { "type": "reference",   "name": "prevOutput",  "value": "previousComponent.outputKey" }
        ],
        "output": [
          { "type": "reference", "name": "resultId", "value": "resultId", "description": "ID of created resource" }
        ]
      }
    ]
  }
}
```

### Variable type reference

| `type` | Source | Notes |
|---|---|---|
| `value` | Literal string in the template | Static; hardcoded |
| `environment` | `process.env[value]` | `default` used when env var is unset |
| `secret` | Secret manager (AWS / MongoDB CSFLE) | Requires secret module loaded |
| `reference` | Output of a previous component | Dot-notation path into previous outputs |
| `protected` | Protected/encrypted value | Advanced use case |
