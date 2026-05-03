---
name: Kozen Engine — Core Concepts
description: >
  Architecture overview of the Kozen framework: the five pillars (Applications, Modules,
  Controllers, Components, Services), the IoC container model based on Awilix, the flow-ID
  tracing convention, module loading lifecycle, shared model interfaces (IConfig, IArgs,
  IModule, IMetadata, IAppType), and the categories system for structured log routing.
category: kozen-engine
tags:
  - kozen
  - architecture
  - IoC
  - dependency-injection
  - module-system
  - KzModule
  - KzController
  - flow-id
---

# Kozen Engine — Core Concepts

## What Kozen is

Kozen is a **modular Task Execution Framework** for Node.js. It provides a uniform runtime
for automation tools, AI integrations, infrastructure pipelines, and CLI utilities. The core
engine (`@kozen/engine`) is intentionally minimal; all domain capabilities are delivered by
independently published modules that plug in at runtime through a `--moduleLoad` flag or an
environment variable.

The engine provides:
- A CLI binary (`kozen`) that dispatches actions to registered controllers
- An MCP (Model Context Protocol) application that exposes module capabilities as AI tools
- An IoC (Inversion of Control) container (built on Awilix) that wires all services
- A structured logging system with console and MongoDB destinations
- Base classes (`KzModule`, `KzController`, `KzApplication`, `KzComponent`, `BaseService`)
  that define the extension contracts every module must satisfy

---

## The five framework pillars

### 1. Applications

An Application is a communication interface between external consumers and the Kozen runtime.
The engine ships two built-in applications:

| Application | Type | How consumers interact |
|---|---|---|
| `CLIApplication` | `cli` | Shell arguments (`--action=module:method`) |
| `MCPApplication` | `mcp` | JSON-RPC over stdio (Model Context Protocol) |

The application type is selected by the `--type` CLI flag or the `KOZEN_APP_TYPE` environment
variable. The default is `cli`. Applications are responsible for starting the IoC container,
loading modules, and dispatching incoming requests to controllers.

### 2. Modules

A Module is a self-contained npm package that registers services, controllers, and components
into the IoC container when loaded. Every module:

- Extends `KzModule` (provided by `@kozen/engine`)
- Declares a unique `metadata.alias` (the routing prefix used in `--action=alias:method`)
- Reads its own `package.json` to populate metadata (name, version, author, license, homepage)
- Implements `register(config, opts)` to return a dependency map keyed by IoC token

Modules are loaded at runtime — not at install time. Loading is triggered by:
- `--moduleLoad=@scope/module-name` CLI flag (comma-separated list)
- `KOZEN_MODULE_LOAD=@scope/module1,@scope/module2` environment variable
- `.env` file entry read via `--envFile=path/to/file.env`

### 3. Controllers

A Controller is the entry point for a specific action within a module. Controllers receive
the already-parsed arguments and call the appropriate service through the IoC container.

Two controller types exist:

| Type | Base class | Runtime | Registered in |
|---|---|---|---|
| CLI Controller | `CLIController` (extends `KzController`) | `config.type === 'cli'` | `cli.json` |
| MCP Controller | `MCPController` | `config.type === 'mcp'` | `mcp.json` |

The `--action` value routes to a controller: `--action=secret:get` means
"resolve the `secret` module's controller and call its `get` method."

### 4. Components

Components are the smallest reusable execution units. They implement `KzComponent` and are
used to build composable pipeline templates. Each component implements four lifecycle methods:
`deploy`, `undeploy`, `validate`, and `status`. Components are auto-registered from the
`src/components/` directory using the IoC `type: "auto"` registration.

Components are primarily used by the engine's own pipeline execution system. Module authors
typically do not need to create components unless they are adding pipeline capabilities.

### 5. Services

Services contain business logic. They extend `BaseService`, which provides constructor
injection of two standard dependencies:

```typescript
export class BaseService {
  protected assistant?: IIoC | null;  // IoC container for resolving other services
  public logger?: ILogger | null;     // Structured logger
}
```

Services are registered in `ioc.json` and resolved on demand from within controllers.

---

## IoC container model

Kozen uses Awilix for dependency injection. The container is exposed as the `IoC` service
(token `"IoC"`) inside every module's constructor and service chain.

### Dependency registration tokens

Each service is identified by a string token. Naming convention used across all modules:

| Token format | Example | What it is |
|---|---|---|
| `module:service` | `secret:manager` | A module-scoped service |
| `module:scram` | `iam-rectification:scram` | A variant of a module service |
| `module:cli` | `trigger:cli` | The CLI controller for a module |
| `module:mcp` | `secret:mcp` | The MCP controller for a module |
| `logger:service` | `logger:service` | The shared logger (built into engine) |
| `IoC` | `IoC` | The container itself |

### Dependency descriptor schema

```typescript
interface IDependency {
  target: string;               // Class name (for type 'class') or token (for 'ref')
  type: 'class' | 'value' | 'function' | 'ref' | 'auto';
  lifetime?: 'singleton' | 'transient' | 'scoped';
  path?: string;                // Relative path to the directory containing the class
  key?: string;                 // IoC token (if different from target)
  args?: JsonValue[];           // Constructor arguments (besides injected dependencies)
  dependencies?: Array<{        // Dependencies injected by the container
    key: string;                // Constructor parameter name (must match constructor)
    target: string;             // Token of the service to inject
    type: 'ref';
  }>;
}
```

Lifetime semantics:
- `singleton` — one instance per IoC container (shared across all calls)
- `transient` — new instance on every `resolve()` call
- `scoped` — one instance per request scope (rarely used in Kozen modules)

### Resolving services

From inside a controller or service with an `assistant` reference:

```typescript
const service = await this.assistant?.resolve<IMyService>('my-module:service');
```

From inside a module constructor (the `KzModule` base exposes `this.assistant`):

```typescript
const logger = await this.assistant?.resolve<ILogger>('logger:service');
```

---

## Key shared interfaces

### IConfig

The runtime configuration object passed to every `module.register()` call:

```typescript
interface IConfig extends IArgs {
  id?: string;           // Unique pipeline/run instance ID
  name?: string;         // Human-readable pipeline name
  engine?: string;       // Engine version requirement (semver range)
  version?: string;      // Config schema version
  type?: IAppType;       // Runtime type: 'cli' | 'mcp' | 'rest' | 'sdk' | 'graphql' | 'rpc'
  description?: string;
  dependencies?: IDependencyMap;
  modules?: {
    path?: string;
    mode?: 'inherit' | 'override';
    load?: Array<string | IModuleOpt>;
  };
}
```

The `type` field is the most important: it tells the module which dependency slice to load
(CLI-only controllers, MCP-only controllers, or just core services).

### IArgs

The parsed CLI argument object:

```typescript
interface IArgs {
  help?: string;
  config?: string;       // Path to cfg/config.json
  action: string;        // e.g., "secret:get"
  module?: string;       // Module alias override
  stack?: string;        // Environment: 'dev' | 'test' | 'prod'
  project?: string;      // Project ID for log correlation
  type?: IAppType;
  [key: string]: any;    // All --key=value pairs from CLI
}
```

### IModule

The contract every module must satisfy:

```typescript
interface IModule {
  readonly helper: IIoC | null | undefined;
  metadata?: IMetadata;
  init<T = IArgs>(argv?: string[] | IArgs): Promise<{ args?: T; config?: IConfig | null }>;
  register(config: IConfig | null, opts?: any): Promise<Record<string, IDependency> | null>;
  requires(config: IConfig | null, opts?: any): Promise<Array<string | IModuleOpt> | null>;
  getId(opt?: IConfig): string;
  wait(): Promise<void>;
  log(input: ILogInput, level?: ILogLevel): void;
}
```

### IMetadata

Module self-description, populated from `package.json` in the constructor:

```typescript
interface IMetadata {
  name?: string;         // npm package name (@scope/module-name)
  alias?: string;        // Routing prefix for --action (e.g., 'secret', 'trigger')
  summary?: string;      // Short description
  version?: string;      // Module version
  author?: string;
  license?: string;
  uri?: string;          // Wiki / documentation URL
  methods?: IMethod[];
  properties?: IProperty[];
  [key: string]: any;
}
```

---

## Flow IDs and tracing

Every operation in Kozen is tagged with a `flow` ID — a short string generated from a
timestamp and random component (e.g., `K2025080509533`). The flow ID binds all log entries
for a single operation so you can filter MongoDB logs or console output to reconstruct exactly
what happened during one invocation.

Generate a flow ID with the `getId()` method inherited by all controllers and modules:

```typescript
const flow = this.getId(config);  // or this.getId() for a random ID
```

Pass the ID through every log call:

```typescript
this.logger?.info({
  flow,
  src: 'MyModule:MyService:myMethod',
  message: 'Operation complete',
  data: { result }
});
```

---

## Log categories

Log entries are tagged with a category from the `VCategory` constant (loaded from
`cfg/const.categories.json`). Use the appropriate constant to tag entries for filtering:

```typescript
import { VCategory } from '@kozen/engine';

// Available categories:
VCategory.cli.tool        // 'cli:tool'      — general CLI tool output
VCategory.cli.logger      // 'cli:logger'    — logger module CLI
VCategory.mcp.tool        // 'mcp:tool'      — MCP tool call
VCategory.core.pipeline   // 'core:pipeline' — pipeline engine events
VCategory.cmp.api         // 'cmp:api'       — component API calls
VCategory.test.e2e        // 'test:e2e'      — end-to-end test log
```

---

## Module loading lifecycle

```
CLI invocation: npx kozen --moduleLoad=@kozen/secret --action=secret:get --key=MY_KEY
    ↓
1. KzController.init()       — parse args, load .env file, read cfg/config.json
    ↓
2. CLIApplication.start()    — determine runtime type ('cli')
    ↓
3. For each module in --moduleLoad:
   a. require('@kozen/secret')           — load the npm package
   b. new SecretModule()                 — construct the KzModule subclass
   c. module.register(config)            — get dependency map for runtime type
   d. IoC.registerAll(deps)             — register all services in the container
    ↓
4. Parse --action='secret:get'           — split into alias='secret', method='get'
    ↓
5. IoC.resolve('secret:cli')            — resolve the CLI controller
    ↓
6. controller.get(args)                  — invoke the action method
    ↓
7. controller resolves service from IoC and calls business logic
    ↓
8. logger.wait()                         — flush pending log writes
```
