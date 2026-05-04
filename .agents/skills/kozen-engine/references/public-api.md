---
name: Kozen Engine — Public API
description: >
  Complete reference of every class, interface, and type exported by @kozen/engine. Covers
  the IoC container (IoC class, IIoC interface, all methods), base classes for building
  modules (KzModule, KzController, KzApplication, BaseService, CLIController, MCPController),
  all model interfaces (IConfig, IArgs, IModule, IMetadata, IDependency, IComponent),
  utility exports (getID, readFrom, Env, MdbClient, Logger, JSONT, EnumUtl), and practical
  patterns for using @kozen/engine as a standalone library or as the host platform for new
  modules. Includes CLI and MCP platform integration recipes.
category: kozen-engine
tags:
  - kozen
  - public-api
  - IoC
  - KzModule
  - KzController
  - BaseService
  - IConfig
  - IDependency
  - platform
  - cli-platform
  - mcp-platform
---

# Kozen Engine — Public API

`@kozen/engine` is both a CLI/MCP **runtime** and a **library**. Any project can import from
it to gain access to the IoC container, base classes, interfaces, and utilities — without
running the `kozen` binary. This document covers every publicly exported symbol.

---

## Installation

```bash
npm install @kozen/engine
```

The package exposes a `kozen` binary (for CLI use) and a TypeScript library (for `import`).

---

## Base classes for module development

These are the classes you extend when building a Kozen module.

### `KzModule`

Base class for all Kozen modules. Extend this to package a domain capability.

```typescript
import { KzModule, IConfig, IDependency } from '@kozen/engine';

class MyModule extends KzModule {
  constructor(dependency?: any) {
    super(dependency);
    this.metadata.alias   = 'my-module'; // required — routes --action=my-module:*
    this.metadata.version = '1.0.0';
    this.metadata.summary = 'What this module does';
  }

  public register(
    config: IConfig | null
  ): Promise<Record<string, IDependency> | null> {
    const dep = config?.type === 'cli' ? { ...ioc, ...cli } : ioc;
    return Promise.resolve(this.fix(dep) as Record<string, IDependency>);
  }
}
```

**Inherited properties:**
| Property | Type | Description |
|---|---|---|
| `metadata` | `IMetadata` | Module self-description (alias, version, summary, author, uri) |
| `assistant` | `IIoC \| null` | IoC container injected at construction |

**Inherited methods:**
| Method | Signature | Description |
|---|---|---|
| `fix(dep)` | `(dep: IDependencyMap) => IDependencyMap` | Resolves all relative `path` fields to absolute paths based on `__dirname`. **Must be called before returning from `register()`**. |
| `register(config, opts?)` | `(IConfig \| null, any?) => Promise<Record<string, IDependency> \| null>` | Override to return the dependency map for the given runtime type. |
| `requires(config, opts?)` | `(IConfig \| null, any?) => Promise<Array<string \| IModuleOpt> \| null>` | Override to declare modules that must be loaded before this one. |
| `init(argv?)` | `(string[] \| IArgs) => Promise<{args?, config?}>` | Called by the engine during startup. Override for custom init logic. |
| `getId(opt?)` | `(IConfig?) => string` | Generates a unique flow ID (timestamp + random). |
| `wait()` | `() => Promise<void>` | Waits for pending async logger writes to flush. |
| `log(input, level?)` | `(ILogInput, ILogLevel?) => void` | Logs a structured entry through the injected logger. |

---

### `KzController`

Base class for all CLI and MCP controllers. Provides argument access, IoC resolution, and logging.

```typescript
import { KzController, IArgs, VCategory } from '@kozen/engine';

class MyController extends KzController {
  // Injected by Awilix via constructor parameter names:
  // this.assistant — IoC container
  // this.logger    — ILogger instance
  // this.args      — parsed CLI arguments (after fill() runs)

  public async myAction(): Promise<void> {
    const flow = this.getId(this.args as any);
    await this.log({ flow, src: 'MyController:myAction', message: 'Running' });
    const svc = await this.assistant?.resolve<MyService>('my-module:service');
    const result = await svc?.execute(this.args?.key);
    console.log(JSON.stringify(result));
  }

  public async fill(args: string[] | IArgs): Promise<IArgs> {
    const parsed = await super.fill(args);
    parsed['key'] = parsed['key'] || process.env.MY_KEY;
    return parsed;
  }
}
```

**Inherited properties:**
| Property | Type | Description |
|---|---|---|
| `assistant` | `IIoC \| null` | IoC container — call `resolve<T>(token)` to get services |
| `args` | `any` | The parsed and normalised CLI arguments object |
| `logger` | `ILogger \| null` | Structured logger |
| `srvFile` | `FileService \| null` | File service for reading help `.txt` files |

**Inherited methods:**
| Method | Signature | Description |
|---|---|---|
| `fill(args)` | `(string[] \| IArgs) => Promise<IArgs>` | Parses and normalises arguments. Override to add env var fallbacks. Always call `super.fill()` first. |
| `configure(args)` | `(IArgs) => Promise<IConfig \| null>` | Loads `cfg/config.json` (or `--config` path) and merges with defaults. |
| `init(argv?)` | `(string[] \| IArgs?) => Promise<{args?, config?}>` | Full initialisation: parse args → load config. |
| `load(configPath)` | `(string) => Promise<IConfig>` | Reads and parses a JSON config file. |
| `getId(opt?)` | `(IConfig?) => string` | Generates a flow ID. |
| `log(input, level?)` | `(ILogInput, ILogLevel?) => Promise<void>` | Structured log entry. |
| `wait()` | `() => Promise<void>` | Flush pending logger writes. |
| `help(info?)` | `(IHelpInfo?) => Promise<void>` | Renders the help text from `srvFile`. |
| `extract(argv?)` | `(string[] \| IArgs?) => Record<string, any>` | Protected: parses raw `process.argv` into a key-value map. |

---

### `CLIController`

Abstract class extending `KzController`. Use as the base for CLI controllers instead of
`KzController` when you want the CLI-specific method set. The only difference is context
typing — use either; the engine accepts both.

```typescript
import { CLIController } from '@kozen/engine';

export class TriggerCLIController extends CLIController {
  public async start(): Promise<void> { ... }
  public async help():  Promise<void> { ... }
}
```

---

### `MCPController`

Abstract class extending `KzController`. Adds the mandatory `register(server)` abstract method.

```typescript
import { MCPController } from '@kozen/engine';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';

export class MyMCPController extends MCPController {
  public async register(server: McpServer): Promise<void> {
    server.registerTool('kozen_my_action', {
      description: 'What this tool does for the LLM.',
      inputSchema: {
        key: z.string().describe('The target key.')
      }
    }, async (args) => {
      const svc = await this.assistant?.resolve<MyService>('my-module:service');
      const result = await svc?.execute(args.key);
      return { content: [{ type: 'text', text: JSON.stringify(result) }] };
    });
  }
}
```

**Required override:**
- `register(server: McpServer): Promise<void>` — called once at MCP server startup. Register
  all tools here. Each tool gets a unique name, a Zod input schema, and an async handler.

---

### `KzApplication`

Base class for custom application types. The built-in `CLIApplication` and `MCPApplication`
extend this. Only implement if you are adding a new transport (e.g., REST, WebSocket).

```typescript
import { KzApplication, IConfig, IModule, IArgs } from '@kozen/engine';

class MyApplication extends KzApplication {
  async init(config: IConfig, app: IModule): Promise<void> {
    await super.init(config, app);
    // custom initialisation
  }
  async start(args?: IArgs): Promise<void> {
    // custom dispatch loop
  }
}
```

---

### `BaseService`

Abstract base class for all service implementations. Provides constructor injection of
`assistant` (IoC) and `logger`.

```typescript
import { BaseService } from '@kozen/engine';

export class MyService extends BaseService {
  // this.assistant — IIoC, resolve other services lazily
  // this.logger    — ILogger, structured logging

  async doWork(input: string): Promise<string> {
    this.logger?.info({ src: 'MyService:doWork', message: 'Processing', data: { input } });
    const otherSvc = await this.assistant?.resolve<OtherService>('other:service');
    return await otherSvc?.process(input) ?? '';
  }
}
```

**Constructor:**
```typescript
constructor(dependency?: { assistant: IIoC; logger: ILogger })
```

**Properties:**
| Property | Type | Description |
|---|---|---|
| `assistant` | `IIoC \| null` | IoC container — for lazy service resolution |
| `logger` | `ILogger \| null` | Structured logger |

---

## IoC container

### `IoC` class

The dependency injection container. Backed by Awilix. Accessible inside modules and services
as `this.assistant`. Can also be instantiated standalone.

```typescript
import { IoC } from '@kozen/engine';

const container = new IoC();

await container.register({
  'my:service': {
    target: 'MyService',
    type: 'class',
    lifetime: 'singleton',
    path: './services',
    dependencies: [
      { key: 'assistant', target: 'IoC', type: 'ref' }
    ]
  }
});

const svc = await container.resolve<MyService>('my:service');
```

**Constructor:**
```typescript
constructor(logger?: Console)
```

**Methods:**

| Method | Signature | Description |
|---|---|---|
| `register(dependencies)` | `(IDependency[] \| IDependencyMap) => Promise<void>` | Registers one or many dependencies. Accepts either an array or a key-value map. Supports nested `dependencies` recursively. |
| `get<T>(key)` | `(string \| IDependency) => Promise<T \| null>` | Safe resolution — returns `null` if not found instead of throwing. Will attempt auto-registration if `regex`-based rules are configured. |
| `resolve<T>(key)` | `(string) => Promise<T>` | Resolves a dependency by token. Throws if not found and auto-registration fails. |
| `resolveSync<T>(key)` | `(string) => T` | Synchronous resolution. Does not support auto-registration. Use only when async is unavailable. |
| `unregister(keys)` | `(string[]) => void` | Removes registrations by token. Useful for test teardown or dynamic reconfiguration. |

**Properties:**

| Property | Type | Description |
|---|---|---|
| `logger` | `Console \| null` | Console logger for internal IoC messages |
| `store` | `IDependencyMap` | Raw registration map (key → IDependency descriptor) |
| `map` | `IDependencyClassMap` | Module-grouped dependency map |
| `config` | `IDependencyMap` (readonly) | Alias for `store` |

### `IIoC` interface

The interface that `this.assistant` exposes inside modules and services. All methods match
the `IoC` class above. Use `IIoC` in TypeScript type declarations to avoid circular imports.

---

## Model interfaces

### `IConfig`

The runtime configuration object. Passed to `module.register()` and available throughout the
framework. The `type` field is the primary signal for conditional IoC overlay loading.

```typescript
interface IConfig extends IArgs {
  id?: string;           // Unique run instance ID (auto-generated)
  name?: string;         // Human-readable pipeline name
  engine?: string;       // Semver range: minimum engine version required
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

### `IArgs`

The parsed CLI argument object. All `--key=value` flags from the command line are merged in,
plus environment variable fallbacks applied by `KzController.fill()`.

```typescript
interface IArgs {
  help?:    string;
  config?:  string;    // --config path to cfg/config.json
  action:   string;    // e.g., 'secret:get'
  module?:  string;    // module alias override
  stack?:   string;    // 'dev' | 'test' | 'prod'
  project?: string;    // log correlation ID
  type?:    IAppType;
  [key: string]: any;  // all custom --key=value pairs
}
```

### `IModule`

The full contract every `KzModule` subclass must satisfy. Use this interface in test code
or adapter layers:

```typescript
interface IModule {
  readonly helper:  IIoC | null | undefined;
  metadata?:        IMetadata;
  init<T>(argv?):   Promise<{ args?: T; config?: IConfig | null }>;
  register(config, opts?): Promise<Record<string, IDependency> | null>;
  requires(config, opts?): Promise<Array<string | IModuleOpt> | null>;
  getId(opt?):      string;
  wait():           Promise<void>;
  log(input, level?): void;
}
```

### `IMetadata`

Module and component self-description. Populated from `package.json` in module constructors.
Also used by `IComponent` to describe inputs, outputs, and properties.

```typescript
interface IMetadata {
  name?:          string;     // npm package name
  alias?:         string;     // action routing prefix (e.g., 'secret')
  summary?:       string;     // one-line description
  version?:       string;     // semver string
  author?:        string;
  license?:       string;
  uri?:           string;     // docs/wiki URL
  methods?:       IMethod[];
  properties?:    IProperty[];
  dependencies?:  Array<string | IDependency>;
  value?:         any;
  range?:         Array<any>;
  type?:          IStructType;
  default?:       any;
  format?:        string;
  src?:           IDependency;
  [key: string]:  any;        // custom fields allowed
}
```

### `IComponent`

Interface for pipeline component implementations. Components are the building blocks of
declarative JSON pipeline templates:

```typescript
interface IComponent {
  id?:           string;
  name?:         string;
  description?:  string;
  version?:      string;
  engine?:       string;      // minimum engine version
  orchestrator?: string;      // e.g., 'pulumi', 'terraform', 'node'
  output?:       Array<IComponentOutput> | Record<string, IComponentOutput>;
  input?:        Array<IComponentInput>  | Record<string, IComponentInput>;
  setup?:        Array<IMetadata>        | Record<string, IMetadata>;
  dependency?:   Array<IMetadata>        | Record<string, IMetadata>;
  [key: string]: any;
}

type IComponentInput  = IMetadata;
type IComponentOutput = IMetadata;
```

### `IModuleOpt`

Options object for specifying a module in a `modules.load` array with custom path or alias:

```typescript
interface IModuleOpt {
  key?:   string;   // IoC token for the module
  path?:  string;   // Path to the module directory (for local modules)
  alias?: string;   // Action routing prefix override
  name?:  string;   // npm package name
}
```

---

## IDependency and related types

(See `configuration.md` for the full field-by-field reference and examples.)

```typescript
type IDependencyType =
  | 'class' | 'value' | 'function' | 'method' | 'action'
  | 'alias' | 'ref' | 'auto' | 'instance' | 'object';

type IDependencyLifetime = 'singleton' | 'transient' | 'scoped';

type IModuleType = 'esm' | 'mjs' | 'cjs' | 'module' | 'commonjs';

type IDependencyMap  = { [key: string]: IDependency };
type IDependencyList = IDependency[];

interface IDependency {
  key?:          string;
  target?:       any;
  regex?:        string;
  type?:         IDependencyType;
  as?:           IDependencyType;
  lifetime?:     IDependencyLifetime;
  moduleType?:   IModuleType;
  template?:     string;
  path?:         string;
  file?:         string;
  args?:         IJSON[] | any[];
  dependencies?: IDependencyList | IDependencyMap;
  category?:     string;
  raw?:          boolean;
}
```

---

## Utility exports

### `getID()`

Generates a unique, human-readable flow identifier from a timestamp and random suffix.
The format is `K` + digits (e.g., `K20260504143205`). Used for distributed tracing.

```typescript
import { getID } from '@kozen/engine';

const flow = getID(); // 'K20260504143205'
```

Also available as `this.getId()` from any `KzController` or `KzModule`.

### `readFrom(path)`

Reads the text content of a file. Used by the engine to load config and help files.

```typescript
import { readFrom } from '@kozen/engine';

const content = await readFrom('./cfg/config.json');
```

### `Env`

Environment variable manager. Reads from `process.env` with type coercion and defaults.

```typescript
import { Env } from '@kozen/engine';

const uri    = Env.get('MDB_URI', 'mongodb://localhost:27017');
const debug  = Env.getBoolean('KOZEN_DEBUG', false);
const level  = Env.get('KOZEN_LOG_LEVEL', 'DEBUG');
```

### `MdbClient` and `IMdbClientOpt`

Managed MongoDB client with connection pooling and options normalisation.

```typescript
import { MdbClient, IMdbClientOpt } from '@kozen/engine';

const opts: IMdbClientOpt = {
  uri:        process.env.MDB_URI!,
  database:   'my-db',
  collection: 'my-collection'
};

const client = new MdbClient(opts);
const db     = await client.getDb();
```

### `Logger` and `ILogLevel`

The engine's structured logger class. Normally injected via IoC as `logger:service`, but
can be instantiated directly for standalone use.

```typescript
import { Logger, ILogLevel } from '@kozen/engine';

const logger = new Logger({ console: { enabled: true, level: 'DEBUG' } });

logger.info({ src: 'MyApp:main', message: 'Starting', data: { version: '1.0.0' } });
logger.warn({ src: 'MyApp:main', message: 'Deprecated option used' });
logger.error({ src: 'MyApp:main', message: 'Unhandled error', data: { error } });
```

### `JSONT`

JSON utility instance with helpers for safe parsing, deep merge, and path-access.

```typescript
import { JSONT } from '@kozen/engine';

const merged  = JSONT.merge(baseConfig, overrideConfig);
const safeVal = JSONT.get(obj, 'nested.key.path', defaultValue);
```

### `EnumUtl`

Enum utility class for converting between enum keys and values.

```typescript
import { EnumUtl } from '@kozen/engine';

const level = EnumUtl.fromKey(ILogLevel, 'DEBUG'); // → ILogLevel.DEBUG
```

### `VCategory`

Constant object with pre-defined log category strings. Import to tag log entries for
filtered queries in MongoDB log storage:

```typescript
import { VCategory } from '@kozen/engine';

// VCategory.cli.tool        → 'cli:tool'
// VCategory.mcp.tool        → 'mcp:tool'
// VCategory.core.pipeline   → 'core:pipeline'
// VCategory.cmp.api         → 'cmp:api'
// VCategory.test.e2e        → 'test:e2e'
```

---

## Using @kozen/engine as a platform

### Pattern 1: Kozen CLI as a command router for your project

Any project can use Kozen as a ready-made CLI host without writing argument parsing
infrastructure. Install the engine, write a module, and you get:

- Structured argument parsing (`--key=value` flags + env var fallbacks)
- Help text from `.txt` files
- Structured logging to console and MongoDB
- Configuration file support (`cfg/config.json`)
- Multi-environment support (`--stack=prod`)

```bash
# Install globally
npm install -g @kozen/engine @scope/my-module

# Run any action from any module
kozen --moduleLoad=@scope/my-module --action=my-module:execute --key=value

# Or via .env file (no CLI flags needed)
echo "KOZEN_MODULE_LOAD=@scope/my-module
KOZEN_ACTION=my-module:execute
MY_KEY=value" > .env
kozen
```

### Pattern 2: MCP platform — expose your services to LLMs

By writing an MCP controller, any Kozen module becomes accessible to Claude, GPT-4, and
any other MCP-compatible LLM client without any HTTP server or API design work:

```bash
# Start as MCP server (Claude Desktop / VS Code)
npx kozen --type=mcp --moduleLoad=@scope/my-module

# Or pre-installed variant (for Claude Desktop config)
npx -y @scope/my-module-mcp
```

VS Code / Claude Desktop configuration:
```json
{
  "mcpServers": {
    "my-module": {
      "command": "npx",
      "args": ["-y", "@kozen/engine"],
      "env": {
        "KOZEN_APP_TYPE": "mcp",
        "KOZEN_MODULE_LOAD": "@scope/my-module",
        "MY_KEY": "${env:MY_KEY}"
      }
    }
  }
}
```

### Pattern 3: Library use — IoC container without the CLI

Use `@kozen/engine` purely as a dependency injection framework in any Node.js application:

```typescript
import { IoC, BaseService, KzModule } from '@kozen/engine';

// Set up a container
const container = new IoC();

await container.register({
  'app:db': {
    target: 'DatabaseService',
    type: 'class',
    lifetime: 'singleton',
    path: './services',
    dependencies: [
      { key: 'assistant', target: 'IoC', type: 'ref' }
    ]
  },
  'app:api': {
    target: 'ApiService',
    type: 'class',
    lifetime: 'transient',
    path: './services',
    dependencies: [
      { key: 'assistant', target: 'IoC', type: 'ref' },
      { key: 'db',        target: 'app:db', type: 'ref' }
    ]
  }
});

// Resolve and use
const api = await container.resolve<ApiService>('app:api');
await api.fetchData();
```

### Pattern 4: Embedding Kozen modules in an existing project

Import a Kozen module's exported services directly (bypassing the CLI runtime) for use in
any Node.js application. This works because every module exports its service classes:

```typescript
// @kozen/secret used as a library — no kozen CLI needed
import { SecretManager } from '@kozen/secret';

const mgr = new SecretManager({ type: 'aws' });
const apiKey = await mgr.resolve('MY_API_KEY');
```

```typescript
// @kozen/trigger used programmatically
import { ChangeStreamService, ITriggerDelegate } from '@kozen/trigger';

const svc = new ChangeStreamService();

const delegate: ITriggerDelegate = {
  insert: async (change, tools) => {
    console.log('New document:', change.fullDocument);
  }
};

await svc.start({
  mdb: { uri: process.env.MDB_URI!, database: 'mydb', collection: 'orders' },
  opt: { target: delegate, type: 'value' }
});
```

---

## Quick reference: what to import for each task

| Task | Import |
|---|---|
| Build a new Kozen module | `KzModule, IConfig, IDependency` |
| Build a CLI controller | `KzController` or `CLIController`, `IArgs, VCategory` |
| Build an MCP controller | `MCPController` |
| Build a service | `BaseService` |
| Use the IoC container standalone | `IoC, IDependency, IDependencyMap` |
| Describe a module's interface | `IModule, IMetadata, IModuleOpt` |
| Describe a service's interface | `IIoC, IConfig, IArgs` |
| Build a pipeline component | `IComponent, IComponentInput, IComponentOutput` |
| Generate a flow/trace ID | `getID` or `this.getId()` |
| MongoDB utility | `MdbClient, IMdbClientOpt` |
| Standalone logging | `Logger, ILogLevel` |
| Environment variables | `Env` |
| JSON utilities | `JSONT` |
| Log categories | `VCategory` |
