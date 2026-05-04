---
name: Kozen Engine — Core Concepts
description: >
  Architecture overview of the Kozen framework: the five pillars (Applications, Modules,
  Controllers, Components, Services), deep dive into the IoC container (Awilix, registration
  strategies, lifetimes, fix(), lazy vs eager resolution), when to use each concept and what
  benefits each brings, the flow-ID tracing convention, module loading lifecycle, and shared
  model interfaces (IConfig, IArgs, IModule, IMetadata, IAppType).
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
  - inversion-of-control
  - awilix
---

# Kozen Engine — Core Concepts

## What Kozen is

Kozen is a **modular Task Execution Framework** for Node.js. It provides a uniform runtime
for automation tools, AI integrations, infrastructure pipelines, and CLI utilities. The core
engine (`@kozen/engine`) is intentionally minimal; all domain capabilities are delivered by
independently published modules that plug in at runtime through a `--moduleLoad` flag or
environment variable — the engine never needs to know about them at install time.

The engine provides:
- A CLI binary (`kozen`) that dispatches actions to registered controllers
- An MCP (Model Context Protocol) server that exposes module capabilities as AI tools
- An IoC (Inversion of Control) container (built on Awilix) that wires all services
- A structured logging system with console and MongoDB destinations
- Base classes (`KzModule`, `KzController`, `KzApplication`, `BaseService`) that define
  the extension contracts every module must satisfy

---

## The five framework pillars

### 1. Applications

An **Application** is a communication interface between external consumers and the Kozen
runtime. It manages the full lifecycle: starts the IoC container, triggers module loading,
and dispatches requests to controllers.

| Application | Type string | How consumers interact |
|---|---|---|
| `CLIApplication` | `'cli'` | Shell arguments (`--action=module:method`) |
| `MCPApplication` | `'mcp'` | JSON-RPC over stdio (Model Context Protocol) |

The type is selected by `--type` or `KOZEN_APP_TYPE`. Default is `'cli'`.

**When to extend or create an Application:**
- Almost never — the two built-in applications cover all current use cases.
- Only if you need a new transport protocol (e.g., REST, WebSocket) to expose Kozen actions.

**What an Application does for module authors:**
- It is the reason `register()` receives a `config` with a `type` field. The Application
  initialises before modules load, sets `config.type`, and passes it down. Module authors use
  `config.type` to decide which IoC overlay to merge (`cli.json` vs `mcp.json`).

---

### 2. Modules

A **Module** is a self-contained npm package that adds a domain capability to Kozen. It is
the unit of distribution and the unit of loading. Every module:

- Extends `KzModule` (base class from `@kozen/engine`)
- Declares a unique `metadata.alias` (used as the action prefix, e.g., `secret`, `trigger`)
- Reads its own `package.json` at construction time to populate metadata
- Implements `register(config, opts)` to return a dependency map keyed by IoC token

Modules are loaded at runtime by name:

```bash
npx kozen --moduleLoad=@kozen/secret,@kozen/trigger --action=secret:get --key=MY_KEY
```

**The module is the plugin boundary.** The engine never imports `@kozen/secret` at compile
time — it resolves the package at runtime and constructs the exported `KzModule` subclass.
This is what makes the ecosystem open-ended: anyone can publish a Kozen module and users
load it without changing the engine.

**When to create a Module:**
- When packaging a new domain capability (secret management, IAM validation, data pipelines,
  monitoring, etc.) that should be independently installed and versioned.
- When you want the capability to be accessible from both CLI and MCP without duplicating
  the wiring logic.

**What to NOT put in a Module:**
- Business logic — that goes in Services.
- CLI argument parsing — that goes in CLI Controllers.
- MCP tool registration — that goes in MCP Controllers.
- The module itself only provides the dependency map; it has no business logic of its own.

---

### 3. Controllers

A **Controller** is the adapter between a communication interface (CLI arguments or MCP JSON)
and the services that implement the actual logic. Controllers are thin: they parse inputs,
resolve services from IoC, call service methods, and format outputs.

Two controller types exist:

| Type | Base class | Registered in | Runtime condition |
|---|---|---|---|
| CLI Controller | `KzController` (or `CLIController`) | `cli.json` | `config.type === 'cli'` |
| MCP Controller | `MCPController` | `mcp.json` | `config.type === 'mcp'` |

**Action routing for CLI:**
The `--action` flag has the format `alias:method`. The engine resolves the controller token
`alias:controller:cli` from IoC, then calls `controller.method(args)`.

```
--action=secret:get  →  resolve 'secret:controller:cli'  →  call controller.get(args)
--action=trigger:start  →  resolve 'trigger:controller:cli'  →  call controller.start(args)
```

**MCP tool registration:**
An MCP controller implements the abstract `register(server: McpServer)` method. It is called
once at startup and registers one or more named tools with the MCP server. After that the
server handles inbound tool calls and routes them to the appropriate handler.

**When to implement a CLI Controller:**
- Your module exposes user-facing commands (`start`, `stop`, `get`, `set`, `verify`, `help`).
- You need to parse CLI arguments and environment variable fallbacks.

**When to implement an MCP Controller:**
- Your module should be callable by an AI assistant (Claude, GPT, etc.).
- You want to expose structured tools with Zod-validated schemas and descriptions.

**When you need BOTH:**
- Almost always — if a capability is useful from the CLI it is also useful from an AI tool.
  The shared service layer means no logic duplication; only the input/output adapters differ.

**Key constraint:** A CLI controller is ONLY loaded when `config.type === 'cli'` because it
is defined only in `cli.json`, which is only merged in the CLI branch of `register()`. A
running MCP server never instantiates CLI controllers. This is intentional: the dependency
tree is kept lean for the actual runtime environment.

---

### 4. Components

A **Component** is the smallest reusable execution unit for declarative pipeline templates.
Components implement `KzComponent` and are auto-registered from the `src/components/`
directory using the IoC `type: "auto"` registration pattern.

Each component exposes four lifecycle methods: `deploy`, `undeploy`, `validate`, and `status`.
Pipeline templates reference components by name in a JSON `components` array; the engine
resolves each by name from IoC and calls the appropriate lifecycle method.

**When to create a Component:**
- You are building a new step type for the Kozen pipeline engine (infrastructure provisioning,
  data transformation, external API calls, etc.).
- You want the step to be composable in JSON pipeline templates without writing custom code.

**When NOT to create a Component:**
- For a simple domain operation invoked from CLI or MCP — use a Service + Controller instead.
- Components are pipeline primitives, not general-purpose utilities.

Components are the least commonly extended pillar for module authors. Most modules provide
Modules + Controllers + Services; Components are primarily used by the engine's own
infrastructure pipeline system.

---

### 5. Services

A **Service** contains all business logic. Services extend `BaseService`, which injects two
dependencies via constructor injection:

```typescript
export class BaseService {
  protected assistant?: IIoC | null;  // IoC container — resolve other services lazily
  public logger?: ILogger | null;     // Structured logger
}
```

Services are registered in `ioc.json` with `"lifetime": "singleton"` (for stateless clients)
or `"lifetime": "transient"` (for stateful operations).

**When to create a Service:**
- For any substantive logic: connecting to MongoDB, calling an external API, parsing data,
  orchestrating sub-operations.
- One service class per distinct concern (SecretManagerAWS, SecretManagerMDB, ChangeStreamService).

**When NOT to put logic in a Service:**
- Input validation and argument parsing belong in Controllers.
- IoC wiring configuration belongs in JSON config files.
- Module metadata belongs in the Module constructor.

---

## IoC container — deep dive

The IoC (Inversion of Control) container is the core mechanism that makes Kozen flexible.
Understanding it deeply is essential for building correct modules.

### What problem IoC solves

Without IoC, a CLI controller that needs a logger and a MongoDB client would instantiate
both directly in its constructor. The result:
- The controller is coupled to the specific logger implementation and MongoDB client class.
- Testing requires mocking at the class level.
- Swapping the logger for a different implementation requires editing every class that uses it.
- The module author must know when to create singletons vs transient instances.

With IoC, each class declares what it needs by name in its constructor parameters. The
container resolves those names to actual instances, handling the wiring, lifecycle, and
singleton/transient semantics automatically. The controller only knows the interface; the
container decides the concrete implementation.

### Awilix under the hood

Kozen wraps Awilix in the `IoC` class (`@kozen/engine/src/shared/tools/ioc/IoC.ts`).
Awilix provides:
- **Constructor injection**: Awilix inspects constructor parameter names and injects matching
  registrations. Parameter name must match the registered token key exactly.
- **Proxy cradle**: Resolution on demand — `container.cradle.myService` triggers lazy
  instantiation.
- **Lifetime management**: `singleton`, `transient`, `scoped` per registration.
- **`asClass`, `asFunction`, `asValue`, `aliasTo`**: Factory strategies for each type.

The `IoC` class adds:
- JSON-driven registration (the dependency map from `register()`)
- `fix()` — absolute path resolution for `path` fields
- Dynamic import support (ESM and CJS modules)
- Template-based path resolution (`{path}/{target}`)
- Safe `.get()` with null-return instead of throwing on missing tokens

### The five registration strategies

Every entry in a dependency map specifies a `type` that tells the IoC container how to
instantiate the dependency:

#### `"type": "class"` — instantiate a TypeScript/JavaScript class

The most common type. The container calls `new TargetClass(injectedDeps)`.

```json
{
  "trigger:service": {
    "target": "ChangeStreamService",
    "type": "class",
    "lifetime": "transient",
    "path": "../services",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

The `target` is the **exported class name** (not the filename). The `path` is the directory
containing that class, relative to the compiled file that calls `this.fix()`. The `key` in
each dependency entry must match the constructor parameter name exactly — Awilix uses
parameter name matching.

#### `"type": "ref"` — reference another registered service

Used exclusively inside `dependencies` arrays to inject an already-registered service:

```json
{ "key": "logger", "target": "logger:service", "type": "ref" }
```

This tells the container: "inject the service registered as `logger:service` into the
constructor parameter named `logger`." The `logger:service` token is provided by the engine's
built-in LoggerModule — it is always available.

#### `"type": "value"` — register a static value

No instantiation; the value is stored directly:

```json
{
  "key": "my:config",
  "target": "my:config",
  "type": "value",
  "value": { "endpoint": "https://api.example.com", "timeout": 5000 }
}
```

Use for configuration objects that don't need to be classes.

#### `"type": "function"` / `"method"` — register a factory function

The function is called on resolution and its return value is registered. Used for services
that require asynchronous initialization or factory patterns.

#### `"type": "auto"` — scan a directory and register all exports

Scans all `.js`/`.ts` files in the given directory and registers every exported class
automatically. Used for bulk-registering pipeline components:

```json
{
  "path": "../../components",
  "type": "auto",
  "lifetime": "singleton",
  "dependencies": [
    { "key": "assistant", "target": "IoC",            "type": "ref" },
    { "key": "logger",    "target": "logger:service", "type": "ref" }
  ]
}
```

### Lifetime management

The `lifetime` field controls how many instances the container creates:

| Lifetime | Instances created | Use when |
|---|---|---|
| `singleton` | One per IoC container, shared across all resolutions | Stateless services, connection pools, the logger, shared API clients |
| `transient` | New instance on every `resolve()` call | Stateful operations that hold per-request state (e.g., a ChangeStreamService that keeps a MongoDB cursor open) |
| `scoped` | One per scope (e.g., one per HTTP request) | Rarely used in Kozen modules |

**Rule of thumb:** Start with `singleton` unless the service maintains state between calls
that must be isolated per invocation. The `ChangeStreamService` in `@kozen/trigger` is
`transient` because each `start()` call owns its own `ChangeStream` and `MongoClient`.
The `SecretManager` in `@kozen/secret` is `singleton` because it is stateless.

### The `fix()` function — why it is mandatory

`KzModule.fix(dep)` iterates the entire dependency map and converts every relative `path`
string to an absolute path using `__dirname` of the compiled module file.

**Why this matters:** When a module is published to npm and installed as a `node_modules`
package, the relative path `../services` in `ioc.json` is relative to the compiled `dist/`
file that imports it. But the IoC container uses `require()` or dynamic `import()` at
resolution time, and the current working directory (`process.cwd()`) at that point is the
host project's root — not the module's `dist/` directory. Without `fix()`, every path would
resolve to the wrong location and the container would throw `Cannot find module`.

Always call `this.fix(dep)` and always do it before returning from `register()`:

```typescript
dep = this.fix(dep);
return Promise.resolve(dep as Record<string, IDependency>);
```

### Resolving services from within controllers and services

**Eager injection via constructor (preferred for always-needed services):**

The `dependencies` array in the JSON config does this automatically. The container injects
the service when it instantiates the controller:

```typescript
export class TriggerCLIController extends KzController {
  protected srvTrigger?: ChangeStreamService | null;

  constructor(dependency?: {
    assistant: IIoC;
    logger: ILogger;
    srvTrigger?: ChangeStreamService;
    srvFile?: FileService;
  }) {
    super(dependency);
    this.srvTrigger = dependency?.srvTrigger ?? null;
  }
}
```

This is wired in `cli.json`:
```json
{
  "trigger:controller:cli": {
    ...
    "dependencies": [
      { "key": "srvTrigger", "target": "trigger:service", "type": "ref" }
    ]
  }
}
```

**Lazy resolution via `this.assistant?.resolve()` (preferred for conditionally-needed services):**

Use when you only need the service for specific actions, or when the service may not be
registered in all runtime configurations:

```typescript
public async execute(): Promise<void> {
  const service = await this.assistant?.resolve<IMyService>('my-module:service');
  const result = await service?.execute(this.args);
}
```

**Rule:** Prefer constructor injection for services the controller always uses. Prefer lazy
resolution when the service is heavy (e.g., opens a database connection) and you want to
avoid instantiating it unnecessarily, or when you need the flexibility to resolve different
implementations at call time.

### How IoC enables module flexibility

The IoC container is what makes the following possible without changing the engine:

1. **Replacing a service implementation:** Register a different class under the same token.
   Any controller resolving `secret:manager` gets the new implementation automatically.

2. **Loading modules conditionally:** The engine calls `module.register(config)` for each
   loaded module. Each module returns only the dependencies it wants to register for that
   runtime type. If a module is not loaded, its services simply do not exist in the container.

3. **Composing modules:** One module can resolve services from another module:
   ```typescript
   // Inside @kozen/trigger, a delegate can resolve @kozen/secret via ITriggerTools
   const secretManager = await tools.assistant?.resolve<ISecretManager>('secret:manager');
   const apiKey = await secretManager?.resolve('MY_API_KEY');
   ```
   This works because both modules are loaded into the same IoC container. The trigger module
   does not import the secret module at compile time; it only knows the token string and the
   interface.

4. **Overriding core engine services:** A module can register its own `logger:service` token,
   and every other module that resolves `logger:service` will get the override. This is how
   custom log processors are injected without modifying the engine.

5. **Testing with stubs:** In tests, register a stub under the same token before resolving.
   The class under test receives the stub automatically through the constructor.

---

## Key shared interfaces

### IConfig

The runtime configuration object passed to every `module.register()` call:

```typescript
interface IConfig extends IArgs {
  id?: string;           // Unique pipeline/run instance ID (auto-generated if absent)
  name?: string;         // Human-readable pipeline name
  engine?: string;       // Engine version requirement (semver range)
  version?: string;      // Config schema version
  type?: IAppType;       // 'cli' | 'mcp' | 'rest' | 'sdk' | 'graphql' | 'rpc'
  description?: string;
  dependencies?: IDependencyMap;
  modules?: {
    path?: string;
    mode?: 'inherit' | 'override';
    load?: Array<string | IModuleOpt>;
  };
}
```

The `type` field is the most critical for module authors. It determines which overlay config
is merged in `register()`. When `type` is absent (SDK / programmatic use), only `ioc.json`
is merged — core services only, no controllers.

### IArgs

The parsed CLI argument object — the full set of `--key=value` flags passed to `kozen`,
augmented with environment variable fallbacks by `KzController.fill()`:

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

The full contract every module must satisfy:

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

The `requires()` method (rarely overridden) declares other modules that must be loaded before
this one. The framework resolves these transitively before calling `register()`.

### IMetadata

Module self-description, populated from `package.json` in the constructor:

```typescript
interface IMetadata {
  name?: string;         // npm package name (@scope/module-name)
  alias?: string;        // Routing prefix for --action (e.g., 'secret', 'trigger')
  summary?: string;      // Short description (from package.json description)
  version?: string;      // Module version (from package.json version)
  author?: string;       // Author (from package.json author)
  license?: string;      // License (from package.json license)
  uri?: string;          // Wiki / documentation URL (from package.json homepage)
  methods?: IMethod[];
  properties?: IProperty[];
  [key: string]: any;
}
```

---

## IoC dependency token naming conventions

Every service is identified by a string token. The convention used across all Kozen modules:

| Pattern | Example | What it represents |
|---|---|---|
| `alias:service` | `secret:manager` | A module's main service (singleton typical) |
| `alias:variant` | `iam-rectification:scram` | A named variant of a module service |
| `alias:controller:cli` | `trigger:controller:cli` | The CLI controller for a module |
| `alias:controller:mcp` | `secret:controller:mcp` | The MCP controller for a module |
| `logger:service` | `logger:service` | The shared structured logger (built into engine) |
| `IoC` | `IoC` | The container itself (always available) |

**Why these tokens matter:** When the engine dispatches `--action=secret:get`, it resolves
`secret:controller:cli` from IoC (the alias is `secret`, the type is `cli`). If your module
registers under a different token format, action routing breaks. Always follow the convention.

---

## Flow IDs and distributed tracing

Every operation in Kozen is tagged with a `flow` ID — a short alphanumeric string generated
from a timestamp and random component (e.g., `K2025080509533`). The flow ID binds all log
entries for a single operation, enabling reconstruction of the full execution trace by
filtering on one ID.

Generate a flow ID with `getId()`, inherited by all controllers and modules:

```typescript
const flow = this.getId(config);  // deterministic from config ID
// or
const flow = this.getId();        // random, for standalone operations
```

Pass the flow through every log call:

```typescript
this.logger?.info({
  flow,
  src: 'MyModule:MyService:myMethod',
  message: 'Operation complete',
  data: { result }
});
```

The `src` field follows the pattern `ModuleName:ClassName:methodName`. It appears in every
log entry and is the first thing to check when debugging a failure.

---

## Log categories

Log entries are tagged with a category from `VCategory` (loaded from
`cfg/const.categories.json`). Categories enable filtered queries in MongoDB log storage:

```typescript
import { VCategory } from '@kozen/engine';

VCategory.cli.tool        // 'cli:tool'      — general CLI tool output
VCategory.cli.logger      // 'cli:logger'    — logger module CLI
VCategory.mcp.tool        // 'mcp:tool'      — MCP tool call
VCategory.core.pipeline   // 'core:pipeline' — pipeline engine events
VCategory.cmp.api         // 'cmp:api'       — component API calls
VCategory.test.e2e        // 'test:e2e'      — end-to-end test log
```

---

## Module loading lifecycle

The full sequence from CLI invocation to action execution:

```
$ npx kozen --moduleLoad=@kozen/secret --action=secret:get --key=MY_KEY
    │
    ├─ 1. bin/kozen.ts starts
    │     KzController.init()  → parse args, load .env, read cfg/config.json
    │     Outputs: args = { action: 'secret:get', type: 'cli', key: 'MY_KEY', ... }
    │             config = { type: 'cli', dependencies: [...], modules: {...} }
    │
    ├─ 2. CLIApplication.start(args)
    │     Confirms config.type === 'cli'
    │
    ├─ 3. Module loading loop — for each module in --moduleLoad:
    │     a. require('@kozen/secret')        ← dynamic import of the npm package
    │     b. new SecretModule()             ← construct the KzModule subclass
    │                                          (reads package.json, sets metadata.alias)
    │     c. module.register(config)        ← returns { 'secret:manager': {...},
    │                                                    'secret:controller:cli': {...} }
    │     d. IoC.registerAll(depMap)        ← registers all dependencies in the container
    │
    ├─ 4. Action dispatch
    │     --action = 'secret:get'
    │     alias = 'secret', method = 'get'
    │     token = 'secret:controller:cli'   ← constructed from alias + config.type
    │
    ├─ 5. IoC.resolve('secret:controller:cli')
    │     Awilix instantiates SecretCLIController with constructor injection:
    │       new SecretCLIController({ assistant: IoC, logger: loggerService })
    │
    ├─ 6. controller.get(args)
    │     Reads args.key, calls this.assistant.resolve('secret:manager'),
    │     calls manager.resolve(key), logs result
    │
    └─ 7. logger.wait()  → flush pending async log writes, then process.exit(0)
```

This lifecycle is identical for MCP, except steps 4–6 are replaced by the MCP server
listening for JSON-RPC tool calls and routing them to `MCPController.register(server)`.
