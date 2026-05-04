---
name: Kozen Module Development
description: >
  Complete step-by-step guide for building a distributable Kozen module from scratch.
  Covers project scaffolding (directory layout, package.json, tsconfig.json), the role and
  internals of the register() function, why IoC configs are split across ioc.json/cli.json/
  mcp.json, why package.json is loaded at runtime, building CLI controllers (argument parsing,
  fill(), action routing, help() with .txt docs), building MCP controllers (server.registerTool,
  Zod schemas, content response format), implementing services (BaseService, logging, error
  handling), inline module documentation structure (.txt files), the full npm publishing
  workflow, and a complete worked example.
category: kozen-engine
tags:
  - kozen
  - module-development
  - KzModule
  - IoC
  - register
  - CLI-controller
  - MCP-controller
  - BaseService
  - npm-publish
  - module-scaffold
  - docs-txt
  - package-json
---

# Kozen Module Development

This guide covers everything needed to build a complete Kozen module from an empty directory
to a published npm package. Use `@kozen/secret`, `@kozen/trigger`, and
`@kozen/iam-rectification` as reference implementations.

---

## 1. Project structure

Every Kozen module follows this layout:

```
@scope/my-module/
├── src/
│   ├── index.ts                      ← module entry point: KzModule subclass + public API
│   ├── controllers/
│   │   ├── MyCLIController.ts        ← CLI controller (if module exposes CLI actions)
│   │   └── MyMCPController.ts        ← MCP controller (if module exposes MCP tools)
│   ├── services/
│   │   └── MyService.ts              ← business logic
│   ├── models/
│   │   └── MyModel.ts                ← TypeScript interfaces for the public API
│   ├── configs/
│   │   ├── ioc.json                  ← always loaded; registers core services
│   │   ├── cli.json                  ← merged when config.type === 'cli'
│   │   └── mcp.json                  ← merged when config.type === 'mcp'
│   └── docs/
│       └── my-module.txt             ← plain-text help displayed by the help() action
├── cfg/
│   └── config.json                   ← optional; used when module runs standalone
├── package.json
├── tsconfig.json
└── README.md
```

---

## 2. package.json

```json
{
  "name": "@scope/my-module",
  "version": "1.0.0",
  "description": "Kozen module that provides [domain] capabilities",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist/*",
    "LICENSE",
    "README.md"
  ],
  "publishConfig": {
    "access": "public"
  },
  "scripts": {
    "build":    "tsc && npm run copy:txt",
    "copy:txt": "copyfiles -u 1 src/**/*.txt dist/",
    "start":    "npx kozen --config=cfg/config.json",
    "dev":      "ts-node node_modules/@kozen/engine/dist/bin/kozen.js",
    "mcp:dev":  "npx -y @modelcontextprotocol/inspector npx -y ts-node node_modules/@kozen/engine/dist/bin/kozen.js --type=mcp",
    "mcp:start":"ts-node node_modules/@kozen/engine/dist/bin/kozen.js --type=mcp"
  },
  "keywords": ["kozen", "mongodb", "my-domain"],
  "author": "Your Name",
  "license": "MIT",
  "engines": { "node": ">=18" },
  "homepage": "https://github.com/your-org/my-module/wiki",
  "bugs":     { "url": "https://github.com/your-org/my-module/issues" },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/your-org/my-module.git"
  },
  "dependencies": {
    "@kozen/engine": "^1.1.15",
    "zod": "^4.1.0"
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "copyfiles": "^2.4.1",
    "ts-node":   "^10.9.0",
    "typescript":"^5.0.0"
  }
}
```

Key rules:
- `@kozen/engine` is a **runtime** dependency — the module's source imports from it at
  runtime, so it must be present when the module executes.
- `"files": ["dist/*"]` limits the published tarball to compiled output only. `src/` is
  never published.
- `publishConfig.access: "public"` is required for scoped packages on the public registry.
- The `copy:txt` script copies `src/**/*.txt` to `dist/` after TypeScript compilation.
  Without this, the `FileService` cannot find the help documentation at runtime.
- The `dev` script runs `kozen.js` via `ts-node`, enabling development without a build step.

### Why package.json is read at module construction time

All Kozen module constructors read their own `package.json` at runtime with `fs.readFileSync`:

```typescript
const pac = JSON.parse(
  fs.readFileSync(path.resolve(__dirname, '../package.json'), 'utf-8')
);
this.metadata.summary = pac.description;
this.metadata.version = pac.version;
this.metadata.author  = pac.author;
this.metadata.license = pac.license;
this.metadata.name    = pac.name;
this.metadata.uri     = pac.homepage;
```

There are five reasons for this pattern:

1. **Single source of truth.** `package.json` is already the authoritative place for name,
   version, description, author, and homepage. Reading it avoids having the same information
   in two places that could drift apart.

2. **Accurate version reporting.** The `help()` action prints the module version. If the
   version were hardcoded in TypeScript, a developer could forget to update it before
   publishing. Reading `package.json` means the running binary always reports the version
   that was actually published.

3. **Ecosystem introspection.** The framework and tooling can enumerate loaded modules and
   their metadata (name, version, author, URI) without importing the module's source. This
   feeds into help banners, MCP server info, and monitoring dashboards.

4. **Author and license attribution.** The `metadata.author` and `metadata.license` fields
   appear in the CLI help output, making it clear who owns and supports the module.

5. **Wiki link for the help command.** `metadata.uri` (from `homepage`) is printed by
   `help()` so users know where to find extended documentation without grepping source code.

Always wrap the read in a `try-catch` and log a warning (not an error) if it fails — the
module should still function even if the metadata cannot be loaded:

```typescript
try {
  const pac = JSON.parse(
    fs.readFileSync(path.resolve(__dirname, '../package.json'), 'utf-8')
  );
  this.metadata.summary = pac.description;
  // ...
} catch (error) {
  this.assistant?.logger?.warn({
    src: 'Module:MyModule',
    msg: `Failed to load package.json metadata: ${(error as Error).message}`
  });
}
```

---

## 3. tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "lib": ["ES2020"],
    "outDir": "dist",
    "rootDir": ".",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

`resolveJsonModule: true` is required so that `import cli from './configs/cli.json'` works
in the module entry point.

---

## 4. The module entry point (src/index.ts)

The entry point has two responsibilities: (1) export the `KzModule` subclass, and
(2) re-export all public interfaces and classes the module exposes as a library.

```typescript
import { IConfig, IDependency, KzModule } from '@kozen/engine';
import cli from './configs/cli.json';
import ioc from './configs/ioc.json';
import mcp from './configs/mcp.json';
import fs   from 'fs';
import path from 'path';

export class MyModule extends KzModule {

  constructor(dependency?: any) {
    super(dependency);
    this.metadata.alias = 'my-module';
    try {
      const pac = JSON.parse(
        fs.readFileSync(path.resolve(__dirname, '../package.json'), 'utf-8')
      );
      this.metadata.summary = pac.description;
      this.metadata.version = pac.version;
      this.metadata.author  = pac.author;
      this.metadata.license = pac.license;
      this.metadata.name    = pac.name;
      this.metadata.uri     = pac.homepage;
    } catch (error) {
      this.assistant?.logger?.warn({ src: 'Module:MyModule', msg: String(error) });
    }
  }

  public register(
    config: IConfig | null,
    opts?: any
  ): Promise<Record<string, IDependency> | null> {
    let dep: Record<string, any> = {};
    switch (config?.type) {
      case 'mcp':
        dep = { ...ioc, ...mcp };
        break;
      case 'cli':
        dep = { ...ioc, ...cli };
        break;
      default:
        dep = ioc;       // SDK / programmatic use: core services only
        break;
    }
    dep = this.fix(dep);
    return Promise.resolve(dep as Record<string, IDependency>);
  }
}

export default MyModule;

// Public library exports — what consumers can import from '@scope/my-module'
export { IMyOptions, IMyResult } from './models/MyModel';
export { MyService }             from './services/MyService';
export { MyCLIController }       from './controllers/MyCLIController';
export { MyMCPController }       from './controllers/MyMCPController';
```

---

## 5. The `register()` function in depth

`register(config, opts)` is the most important method in a Kozen module. It is called once
by the engine during module loading, before any controller or service is instantiated.

### What it does

It returns a **dependency map** — a plain JavaScript object where each key is an IoC token
and each value is an `IDependency` descriptor. The engine passes this map to
`IoC.registerAll()`, which registers all entries in the Awilix container.

### Why it exists as a method and not as a static export

The `register()` method receives the live `config` object from the running engine. This is
the only way the module can know the **runtime type** (`'cli'`, `'mcp'`, or undefined) at the
moment of loading. A static export of a JSON file cannot adapt to runtime context. A method
can.

### Why the switch on `config?.type`

Each branch merges a different set of JSON configs:

```typescript
case 'mcp':  dep = { ...ioc, ...mcp };   // core services + MCP controller
case 'cli':  dep = { ...ioc, ...cli };   // core services + CLI controller
default:     dep = ioc;                  // core services only (library/SDK use)
```

The `...` spread means `mcp.json` or `cli.json` entries are added on top of `ioc.json`.
If both files define the same key, the type-specific file wins.

**Why this separation matters:**
- When running as an MCP server, there is no terminal to parse CLI arguments — registering
  `MyCLIController` would waste memory and could cause issues if it tries to read stdin.
- When running as a CLI, the MCP server never starts — registering `MyMCPController` is
  pointless and the `@modelcontextprotocol/sdk` import might not even be available.
- The `default` case (no type / SDK use) means a downstream project can `import { MyService }
  from '@scope/my-module'` and use it programmatically without pulling in any controllers.

### Why `this.fix(dep)` is the last step before returning

`fix()` walks the dependency map and converts relative `path` strings to absolute paths
anchored to `__dirname` of the compiled module file. This is mandatory for npm-installed
modules because the IoC container resolves paths using Node's `require()` at resolution time,
not at registration time. Without `fix()`, path resolution depends on the caller's working
directory — which is the host project's root, not the module's `dist/` directory — and every
`class` registration fails with `Cannot find module`.

**Never skip `fix()`**. Always apply it to the final merged map before returning.

---

## 6. IoC configuration files — why three files, not one

Most developers ask: "Why not put everything in one `ioc.json`?" The answer is
**conditional loading by runtime type**.

### `src/configs/ioc.json` — always loaded (core services)

Contains the module's core services: everything needed regardless of how the module is used
(CLI, MCP, or library). Typically this is one or more `"lifetime": "singleton"` service
registrations:

```json
{
  "my-module:service": {
    "target": "MyService",
    "type": "class",
    "lifetime": "singleton",
    "path": "../services",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

**Rule:** Only put in `ioc.json` what you need in the `default` case. If a service is only
needed when running as CLI or as MCP, put it in the respective overlay file.

### `src/configs/cli.json` — only loaded for CLI runtime

Contains the CLI controller registration. The CLI controller is the action dispatcher for
`--action=my-module:method` commands. It is never needed in an MCP server or a library:

```json
{
  "my-module:controller:cli": {
    "target": "MyCLIController",
    "type": "class",
    "lifetime": "transient",
    "path": "../controllers",
    "dependencies": [
      { "key": "assistant",  "target": "IoC",            "type": "ref" },
      { "key": "logger",     "target": "logger:service", "type": "ref" },
      { "key": "srvMyModule","target": "my-module:service", "type": "ref" },
      { "key": "srvFile",    "target": "core:file",      "type": "ref" }
    ]
  }
}
```

Note that `cli.json` references `my-module:service` (defined in `ioc.json`) via `"type":
"ref"`. The `{ ...ioc, ...cli }` merge ensures both are registered before Awilix resolves
the controller and injects `srvMyModule`.

**Why `transient` for controllers?** Controllers often hold request-scoped state (the parsed
`args` object). A new instance per dispatch avoids state leaking between CLI invocations if
the process is long-lived (e.g., a daemon or MCP server accidentally loading CLI controllers).

### `src/configs/mcp.json` — only loaded for MCP runtime

Contains the MCP controller registration. The MCP controller calls `server.registerTool()`
once during `MCPApplication.init()`. It is never needed in a CLI run:

```json
{
  "my-module:controller:mcp": {
    "target": "MyMCPController",
    "type": "class",
    "lifetime": "singleton",
    "path": "../controllers",
    "dependencies": [
      { "key": "assistant", "target": "IoC",               "type": "ref" },
      { "key": "logger",    "target": "logger:service",    "type": "ref" }
    ]
  }
}
```

**Why `singleton` for MCP controllers?** The MCP server runs indefinitely. The controller
is instantiated once and the registered tools live for the process lifetime — singletons are
appropriate here.

### Summary: which file for what

| Registration | Goes in | Reason |
|---|---|---|
| Core services (stateless business logic, shared clients) | `ioc.json` | Needed in all runtime modes and as a library |
| CLI controller | `cli.json` | Only exists when `config.type === 'cli'` |
| CLI-only helper services | `cli.json` | Avoids loading heavy services in MCP/SDK contexts |
| MCP controller | `mcp.json` | Only exists when `config.type === 'mcp'` |
| MCP-only helper services | `mcp.json` | Symmetrical to CLI case |

---

## 7. Implementing a Service (src/services/MyService.ts)

```typescript
import { BaseService } from '@kozen/engine';
import { IMyOptions, IMyResult } from '../models/MyModel';

export class MyService extends BaseService {
  // BaseService injects via constructor: this.assistant (IIoC), this.logger (ILogger)

  async execute(options: IMyOptions): Promise<IMyResult> {
    this.logger?.info({
      src: 'MyModule:MyService:execute',
      message: 'Starting execution',
      data: { key: options.key }
    });

    try {
      const result = await this.performWork(options);

      this.logger?.info({
        src: 'MyModule:MyService:execute',
        message: 'Execution complete',
        data: { result }
      });

      return result;
    } catch (error) {
      this.logger?.error({
        src: 'MyModule:MyService:execute',
        message: `Execution failed: ${(error as Error).message}`
      });
      throw error;
    }
  }

  private async performWork(options: IMyOptions): Promise<IMyResult> {
    return { success: true, data: {} };
  }
}
```

Service rules:
- Always extend `BaseService`. Never replicate the constructor injection pattern manually.
- Use `this.logger?.info/warn/error/debug({ src, message, data })` for all output.
- The `src` field pattern is `ModuleName:ClassName:methodName`.
- **Throw errors** from service methods; let controllers decide how to handle and log them.
- Use `this.assistant?.resolve<T>('token')` to lazily resolve other services at call time,
  not in the constructor — the container may not be fully populated at construction time.

---

## 8. Implementing a CLI Controller (src/controllers/MyCLIController.ts)

A CLI controller receives the parsed `IArgs` object and dispatches to services. The
action method name must match the method portion of `--action=alias:method`.

```typescript
import { KzController, IArgs, VCategory } from '@kozen/engine';
import type { MyService } from '../services/MyService';
import type { FileService } from '@kozen/engine';

export class MyCLIController extends KzController {
  protected srvMyModule?: MyService | null;
  protected srvFile?: FileService | null;

  constructor(dependency?: {
    assistant: any;
    logger: any;
    srvMyModule?: MyService;
    srvFile?: FileService;
  }) {
    super(dependency);
    this.srvMyModule = dependency?.srvMyModule ?? null;
    this.srvFile     = dependency?.srvFile     ?? null;
  }

  /** Displays help text read from src/docs/my-module.txt */
  public async help(): Promise<void> {
    const content = await this.srvFile?.read('my-module');
    console.log(content ?? 'No help available.');
  }

  /** Main action: --action=my-module:execute --key=VALUE */
  public async execute(): Promise<void> {
    const flow = this.getId(this.args as any);
    const args = this.args as IArgs & { key?: string; driver?: string };

    await this.log({
      flow,
      src: 'MyModule:MyCLIController:execute',
      message: 'Executing action',
      category: VCategory.cli.tool
    });

    const options = {
      key:    args.key    || process.env.MY_KEY    || '',
      driver: args.driver || process.env.MY_DRIVER || 'default'
    };

    const result = await this.srvMyModule?.execute(options);
    console.log(JSON.stringify(result, null, 2));
  }

  /**
   * Parses and normalises CLI arguments.
   * Called by the engine before the action method is dispatched.
   * Override to apply environment variable fallbacks and type coercions.
   */
  public async fill(args: string[] | IArgs): Promise<IArgs> {
    const parsed = await super.fill(args);
    parsed['key']    = parsed['key']    || process.env.MY_KEY;
    parsed['driver'] = parsed['driver'] || process.env.MY_DRIVER || 'default';
    return parsed;
  }
}
```

Key patterns:
- **Action method = CLI flag method portion.** `--action=my-module:execute` dispatches to
  `MyCLIController.execute()`. The method must be public and async.
- **`fill()` is the normalisation hook.** Override it to apply environment variable fallbacks
  and default values. Always call `await super.fill(args)` first.
- **Argument priority chain:** CLI flag → env var → hardcoded default.
- **`help()` reads from `srvFile`.** The `FileService` looks for a file named by the alias in
  the `docs/` directory. Configure it with `"args": [{ "dir": "./docs" }]` (or the engine
  default). See section 11 for the docs file format.
- **Generate and carry the flow ID** through every log call for distributed tracing.

### Dynamic action naming (IAM-rectification pattern)

When a module has multiple variants of the same action (e.g., `verifySCRAM` and
`verifyX509`), override `fill()` to construct the dynamic method name:

```typescript
public async fill(args: string[] | IArgs): Promise<IArgs> {
  const parsed = await super.fill(args);
  const method = (parsed['method'] || process.env.MY_METHOD || 'SCRAM').toUpperCase();
  // Append method suffix so --action=my-module:verify with --method=SCRAM
  // dispatches to controller.verifySCRAM()
  if (parsed['action'] !== 'help') {
    parsed['action'] = parsed['action'] + method;
  }
  return parsed;
}
```

This pattern avoids needing separate `--action` values for variants of the same operation.

---

## 9. Implementing an MCP Controller (src/controllers/MyMCPController.ts)

MCP controllers register tools with the MCP server during startup. Tool handlers are called
by the LLM client on demand.

```typescript
import { KzController, VCategory } from '@kozen/engine';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';
import type { MyService } from '../services/MyService';

export class MyMCPController extends KzController {

  public async register(server: McpServer): Promise<void> {
    server.registerTool(
      'kozen_mymodule_execute',
      {
        description: 'Execute the main operation of my-module. Returns the result as JSON.',
        inputSchema: {
          key:    z.string().describe('The target key to operate on.'),
          driver: z.string().optional().describe('Backend driver. Default: "default".')
        }
      },
      this.execute.bind(this)
    );
  }

  private async execute(args: { key: string; driver?: string }): Promise<{ content: any[] }> {
    try {
      const service = await this.assistant?.resolve<MyService>('my-module:service');
      const result  = await service?.execute({
        key:    args.key,
        driver: args.driver || process.env.MY_DRIVER || 'default'
      });

      return {
        content: [{ type: 'text', text: JSON.stringify(result, null, 2) }]
      };
    } catch (error) {
      return {
        content: [{ type: 'text', text: `Error: ${(error as Error).message}` }]
      };
    }
  }
}
```

MCP controller rules:
- Tool names follow `kozen_<modulealias>_<action>` to avoid collisions across modules
  loaded simultaneously (e.g., `kozen_secret_select`, `kozen_trigger_start`).
- All `inputSchema` fields use Zod schemas. `.describe()` provides the field description
  shown to the LLM — write descriptions that help the LLM choose the right values.
- Return value is always `{ content: Array<{ type: 'text'; text: string }> }`.
- **Never throw from a tool handler.** Catch errors and return them as content — the MCP
  protocol expects a content response, not an exception. An uncaught error crashes the server.
- Resolve services lazily inside handlers, not in the constructor or `register()` — the
  container may still be loading when `register()` is first called.
- The `register()` method returns `Promise<void>`. Register all tools before returning.

---

## 10. Models file (src/models/MyModel.ts)

Define all public-facing interfaces in a models file. This is what downstream consumers
import when using the module as a library.

```typescript
export interface IMyOptions {
  key: string;
  driver?: string;
}

export interface IMyResult {
  success: boolean;
  data: Record<string, any>;
  error?: string;
}
```

Naming convention: prefix all exported interfaces with `I` (e.g., `IMyOptions`, `IMyResult`).

---

## 11. Inline module documentation (src/docs/my-module.txt)

Every module with a CLI interface should ship a plain-text help file in `src/docs/`. This
file is read and displayed verbatim by the `help()` action via the engine's `FileService`.

**Why `.txt` and not `.md`?** The `FileService` reads and prints the file as-is to stdout.
Markdown markup (`**bold**`, `#` headers) renders as literal characters in a terminal and
degrades the reading experience. Plain text with indentation and spacing is more legible
as CLI output.

**Why a separate file and not a hardcoded string in `help()`?** A file:
- Can be audited, searched, and versioned in git independently of code.
- Is copied to `dist/` by the `copy:txt` build script and is thus part of the published
  package, not source-only.
- Can be read by tooling, documentation generators, and LLMs without executing the code.
- Can be longer than what is comfortable to maintain as a multiline string literal.

### Standard section structure

Follow this structure exactly — every existing module uses it, and the AI skills index on it:

```
<Module Name> — <one-line description>

Description:
  <2-4 sentences: what the module does, what backend it uses, what problem it solves>

Usage:
  kozen [options] --action=<alias>:<action> [module options]
  kozen --module=<alias> --action=<action> [options]

Core Options:
  --stack <name>        Environment name (default: 'dev')
  --project <id>        Project ID for log correlation
  --config <path>       Path to config.json (default: cfg/config.json)
  --module <alias>      Module alias override
  --action <action>     Action to execute

Available Actions:
  <action1>    <Short description of what this action does>
  <action2>    <Short description>
  help         Display this help message

<ModuleName> Options:
  --<param1>       <Description>. Required.
  --<param2>       <Description>. Default: '<default value>'.
  --<param3>       <Description>. Options: value1 | value2.

Environment Variables:
  KOZEN_CONFIG           Path to config.json
  KOZEN_ACTION           Default action
  KOZEN_STACK            Environment name
  KOZEN_PROJECT          Project ID
  MY_MODULE_PARAM1       <Description>
  MY_MODULE_PARAM2       <Description>

Backend Providers:
  <provider1>    <One-line description>
  <provider2>    <One-line description>

Security Features:
  - <Feature 1>
  - <Feature 2>

Examples:
  # <Scenario description>
  kozen --action=my-module:action1 --param1=value

  # <Another scenario>
  kozen --module=my-module --action=action2 --param2=value --stack=prod

  # Using environment variables only
  MY_MODULE_PARAM1=value KOZEN_ACTION=my-module:action1 kozen
```

### Rules for the docs file

- Write in plain text. No markdown symbols.
- Use 2-space indentation for content under section headings.
- Mark required parameters explicitly with "Required." or "(required)".
- Show the default value for every optional parameter.
- Include at least 3 examples covering the most common use cases.
- The file name must match the module alias (e.g., alias `trigger` → `docs/trigger.txt`).
- After every build, verify `dist/docs/my-module.txt` exists. If it doesn't, the `copy:txt`
  script is misconfigured.

### Real examples to reference

- `@kozen/trigger`: `src/docs/trigger.txt` — simple linear structure, 4 examples
- `@kozen/secret`: `src/docs/secret.txt` — multi-action module with backend sections
- `@kozen/iam-rectification`: `src/docs/rectification.txt` — two auth methods with
  separate options blocks for SCRAM vs X.509

---

## 12. Building and testing locally

```bash
# Install dependencies
npm install

# Build (TypeScript + copy .txt files)
npm run build

# Test as CLI (ts-node, no build required during development)
npm run dev -- --action=my-module:help
npm run dev -- --action=my-module:execute --key=test-key

# Test MCP (opens MCP Inspector at localhost:5173)
npm run mcp:dev

# Link locally to test as a node_modules package in another project
npm link
cd /path/to/host-project
npm link @scope/my-module
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

---

## 13. Publishing to npm

```bash
npm run build          # compile + copy .txt files

npm pack --dry-run     # verify tarball contents — must not include src/ or node_modules/

npm publish --access public   # first publish (scoped package)

# Subsequent publishes: bump version first
npm version patch             # 1.0.0 → 1.0.1
npm publish
```

After publishing, verify installation in a clean environment:

```bash
npm install @scope/my-module
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

---

## 14. Complete minimal module example

The smallest valid Kozen module — one service, one CLI controller, no MCP:

**src/configs/ioc.json:**
```json
{
  "greeter:service": {
    "target": "GreeterService",
    "type": "class",
    "lifetime": "singleton",
    "path": "../services",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

**src/configs/cli.json:**
```json
{
  "greeter:controller:cli": {
    "target": "GreeterCLIController",
    "type": "class",
    "lifetime": "transient",
    "path": "../controllers",
    "dependencies": [
      { "key": "assistant",   "target": "IoC",             "type": "ref" },
      { "key": "logger",      "target": "logger:service",  "type": "ref" },
      { "key": "srvGreeter",  "target": "greeter:service", "type": "ref" },
      { "key": "srvFile",     "target": "core:file",       "type": "ref" }
    ]
  }
}
```

**src/services/GreeterService.ts:**
```typescript
import { BaseService } from '@kozen/engine';

export class GreeterService extends BaseService {
  async greet(name: string): Promise<string> {
    this.logger?.info({ src: 'Greeter:GreeterService:greet', message: `Greeting ${name}` });
    return `Hello, ${name}!`;
  }
}
```

**src/controllers/GreeterCLIController.ts:**
```typescript
import { KzController } from '@kozen/engine';
import type { GreeterService } from '../services/GreeterService';

export class GreeterCLIController extends KzController {
  protected srvGreeter?: GreeterService | null;

  constructor(dependency?: { assistant: any; logger: any; srvGreeter?: GreeterService; srvFile?: any }) {
    super(dependency);
    this.srvGreeter = dependency?.srvGreeter ?? null;
  }

  public async greet(): Promise<void> {
    const name = (this.args as any)?.name || 'World';
    const msg  = await this.srvGreeter?.greet(name);
    console.log(msg);
  }

  public async help(): Promise<void> {
    const content = await this.srvFile?.read('greeter');
    console.log(content ?? 'greeter --action=greeter:greet --name=<name>');
  }
}
```

**src/docs/greeter.txt:**
```
greeter — Personalized greeting module for the Kozen framework

Description:
  Provides a configurable greeting action for demonstration and testing.

Usage:
  kozen --action=greeter:greet --name=<name>

Actions:
  greet    Output a greeting for the given name
  help     Display this help message

Options:
  --name   The name to greet. Default: 'World'.

Examples:
  # Basic greeting
  kozen --moduleLoad=@scope/greeter --action=greeter:greet --name=Kozen
  # Output: Hello, Kozen!

  # Default greeting
  kozen --moduleLoad=@scope/greeter --action=greeter:greet
  # Output: Hello, World!
```

**src/index.ts:**
```typescript
import { IConfig, IDependency, KzModule } from '@kozen/engine';
import cli from './configs/cli.json';
import ioc from './configs/ioc.json';
import fs   from 'fs';
import path from 'path';

export class GreeterModule extends KzModule {
  constructor(dependency?: any) {
    super(dependency);
    this.metadata.alias = 'greeter';
    try {
      const pac = JSON.parse(fs.readFileSync(path.resolve(__dirname, '../package.json'), 'utf-8'));
      this.metadata.name    = pac.name;
      this.metadata.version = pac.version;
      this.metadata.summary = pac.description;
      this.metadata.uri     = pac.homepage;
    } catch (_) {}
  }

  public register(config: IConfig | null): Promise<Record<string, IDependency> | null> {
    const dep = config?.type === 'cli' ? { ...ioc, ...cli } : ioc;
    return Promise.resolve(this.fix(dep) as Record<string, IDependency>);
  }
}

export default GreeterModule;
export { GreeterService } from './services/GreeterService';
```

**Usage:**
```bash
npx kozen --moduleLoad=@scope/greeter --action=greeter:greet --name=Kozen
# Output: Hello, Kozen!
```

---

## 15. Checklist before publishing

| Check | Why |
|---|---|
| `this.metadata.alias` is set and unique | Action routing breaks without it |
| `package.json` `homepage` field is set | Printed by `help()` as the docs URL |
| All `ioc.json` `path` values are relative (not absolute) | `fix()` resolves them at runtime |
| `this.fix(dep)` called before returning from `register()` | Absolute paths required for npm-installed modules |
| `src/docs/<alias>.txt` exists and is included in `copy:txt` | `help()` action will fail silently without it |
| Token format follows `alias:controller:cli` / `alias:controller:mcp` | Engine action routing requires this exact format |
| `cli.json` only loaded for `'cli'` branch, `mcp.json` only for `'mcp'` | Prevents registering unused controllers |
| `@kozen/engine` in `dependencies` (not `devDependencies`) | Required at runtime in consuming projects |
| `publishConfig.access: "public"` set | Scoped packages default to private on npm |
| `"files": ["dist/*"]` in `package.json` | Prevents `src/` from being published |
