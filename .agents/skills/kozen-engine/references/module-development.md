---
name: Kozen Module Development
description: >
  Complete step-by-step guide for building a distributable Kozen module from scratch.
  Covers project scaffolding (directory layout, package.json, tsconfig.json), implementing
  KzModule (constructor, register, metadata, fix), defining IoC dependency maps (ioc.json,
  cli.json, mcp.json), building CLI controllers (extending KzController, argument parsing,
  service resolution, logging), building MCP controllers (server.registerTool, Zod schemas,
  content response format), implementing services (extending BaseService, structured logging,
  error handling), the full npm publishing workflow for Kozen modules, and a complete
  worked example of a minimal module.
category: kozen-engine
tags:
  - kozen
  - module-development
  - KzModule
  - IoC
  - CLI-controller
  - MCP-controller
  - BaseService
  - npm-publish
  - module-scaffold
---

# Kozen Module Development

This guide walks through building a complete Kozen module from an empty directory to a
published npm package. Use `@kozen/secret`, `@kozen/trigger`, and `@kozen/iam-rectification`
as reference implementations at all times.

---

## 1. Project structure

Every Kozen module follows this layout:

```
@scope/my-module/
├── src/
│   ├── index.ts                  ← module entry point; exports KzModule subclass + public API
│   ├── controllers/
│   │   ├── MyCLIController.ts    ← CLI controller (if module exposes CLI actions)
│   │   └── MyMCPController.ts    ← MCP controller (if module exposes MCP tools)
│   ├── services/
│   │   └── MyService.ts          ← business logic
│   ├── models/
│   │   └── MyModel.ts            ← TypeScript interfaces exported as public API
│   └── configs/
│       ├── ioc.json              ← always loaded; registers core services
│       ├── cli.json              ← merged on top of ioc.json when type === 'cli'
│       └── mcp.json              ← merged on top of ioc.json when type === 'mcp'
├── cfg/
│   └── config.json               ← optional; used when module is run standalone
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
    "build": "tsc && npm run copy:txt",
    "copy:txt": "copyfiles -u 1 src/**/*.txt dist/",
    "start": "npx kozen --config=cfg/config.json",
    "dev": "ts-node node_modules/@kozen/engine/dist/bin/kozen.js",
    "mcp:dev": "npx -y @modelcontextprotocol/inspector npx -y ts-node node_modules/@kozen/engine/dist/bin/kozen.js --type=mcp",
    "mcp:start": "ts-node node_modules/@kozen/engine/dist/bin/kozen.js --type=mcp"
  },
  "keywords": ["kozen", "mongodb", "my-domain"],
  "author": "Your Name",
  "license": "MIT",
  "engines": { "node": ">=18" },
  "homepage": "https://github.com/your-org/my-module/wiki",
  "bugs": { "url": "https://github.com/your-org/my-module/issues" },
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
    "ts-node": "^10.9.0",
    "typescript": "^5.0.0"
  }
}
```

Key rules:
- `@kozen/engine` is a runtime dependency (not devDependency) — services from the engine
  are imported by the module's source code at runtime.
- `"files": ["dist/*"]` is the only published artifact. The `src/` directory is never published.
- `publishConfig.access: "public"` is required for scoped packages on the public npm registry.
- The `dev` script uses the engine's `kozen.js` binary via `ts-node` so you can develop
  without building first. The `start` script uses the compiled output.

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

The `resolveJsonModule: true` flag is required so that `import cli from './configs/cli.json'`
works in the module entry point.

---

## 4. Implementing the module entry point (src/index.ts)

The entry point has two responsibilities:

1. Export the `KzModule` subclass as both the default export and a named export.
2. Re-export all public interfaces and classes the module exposes as a library.

```typescript
import { IConfig, IDependency, KzModule } from '@kozen/engine';
import cli from './configs/cli.json';
import ioc from './configs/ioc.json';
import mcp from './configs/mcp.json';
import fs from 'fs';
import path from 'path';

export class MyModule extends KzModule {

  constructor(dependency?: any) {
    super(dependency);
    // Set the routing alias used in --action=my-module:method
    this.metadata.alias = 'my-module';
    // Load metadata from package.json for the CLI banner
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
      this.assistant?.logger?.warn({
        src: 'Module:MyModule',
        msg: `Failed to load package.json metadata: ${(error as Error).message}`
      });
    }
  }

  /**
   * Returns the dependency map for the given runtime type.
   * Always merge ioc.json (core services) with the type-specific additions.
   * Always call this.fix() before returning — it resolves relative paths to absolute.
   */
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
        dep = ioc;   // SDK / programmatic use: only core services
        break;
    }
    dep = this.fix(dep);
    return Promise.resolve(dep as Record<string, IDependency>);
  }
}

export default MyModule;

// Public library API — all interfaces and classes consumers may import directly
export { IMyOptions, IMyResult } from './models/MyModel';
export { MyService }            from './services/MyService';
export { MyCLIController }      from './controllers/MyCLIController';
export { MyMCPController }      from './controllers/MyMCPController';
```

### The `this.fix()` call

`KzModule.fix(dep)` iterates over every entry in the dependency map and resolves any `path`
field that is a relative string (`../../services`) to an absolute path based on `__dirname`
of the compiled module file. This is mandatory: without it the IoC container cannot find the
class files at runtime, especially when the module is installed as a node_modules package.

---

## 5. IoC configuration files

### src/configs/ioc.json — core services always registered

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

### src/configs/cli.json — additional registrations for CLI runtime

```json
{
  "my-module:cli": {
    "target": "MyCLIController",
    "type": "class",
    "lifetime": "singleton",
    "path": "../controllers",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

### src/configs/mcp.json — additional registrations for MCP runtime

```json
{
  "my-module:mcp": {
    "target": "MyMCPController",
    "type": "class",
    "lifetime": "singleton",
    "path": "../controllers",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
    ]
  }
}
```

### IDependency type reference

| Field | Values | Meaning |
|---|---|---|
| `type` | `class` | Instantiate the class; requires `path` pointing to its directory |
| `type` | `value` | Register a static value directly |
| `type` | `function` | Call the function and register its return value |
| `type` | `ref` | Reference another registered service by token |
| `type` | `auto` | Scan a directory and auto-register all exported classes |
| `lifetime` | `singleton` | One instance per IoC container |
| `lifetime` | `transient` | New instance on every `resolve()` |
| `path` | relative string | Path relative to the compiled `dist/` file calling `this.fix()` |
| `args` | JSON array | Constructor arguments; `null` entries are skipped |
| `dependencies` | array | Injected into the constructor; `key` must match the parameter name |

---

## 6. Implementing a Service (src/services/MyService.ts)

```typescript
import { BaseService } from '@kozen/engine';
import { IMyOptions, IMyResult } from '../models/MyModel';

export class MyService extends BaseService {
  // BaseService injects: this.assistant (IIoC), this.logger (ILogger)

  /**
   * @param options  Input options
   * @returns        Result of the operation
   */
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
    // Domain logic here
    return { success: true, data: {} };
  }
}
```

Rules for services:
- Always extend `BaseService`. Never implement the `IIoC` injection pattern manually.
- Use `this.logger?.info/warn/error/debug({ src, message, data })` for all output.
- The `src` field follows the pattern `Module:ServiceClass:methodName`.
- Throw errors from service methods; let controllers decide how to handle them.
- Use `this.assistant?.resolve<T>('token')` to lazily resolve other services at call time,
  not in the constructor (the container may not be fully populated yet at construction time).

---

## 7. Implementing a CLI Controller (src/controllers/MyCLIController.ts)

CLI controllers extend `KzController` (or the more specific `CLIController`), which provides
argument parsing (`this.args`), IoC resolution (`this.assistant`), and logging (`this.logger`).

```typescript
import { KzController, IArgs, VCategory } from '@kozen/engine';
import { IMyOptions } from '../models/MyModel';

export class MyCLIController extends KzController {
  // Inherited: this.args (parsed CLI arguments), this.assistant (IoC), this.logger

  /** Shows help text for this module */
  public async help(): Promise<void> {
    console.log(`
my-module — [Short description]

Usage:
  kozen --action=my-module:execute --key=VALUE [--driver=default]

Actions:
  help        Show this help
  execute     Run the main operation

Options:
  --key       Required. The target key.
  --driver    Optional. Backend driver. Default: 'default'.
    `);
  }

  /** Executes the main operation */
  public async execute(): Promise<void> {
    const flow = this.getId(this.args as any);

    await this.log({
      flow,
      src: 'MyModule:MyCLIController:execute',
      message: 'Executing action',
      category: VCategory.cli.tool
    });

    const service = await this.assistant?.resolve<import('../services/MyService').MyService>(
      'my-module:service'
    );

    const options: IMyOptions = {
      key:    this.args?.key    || process.env.MY_KEY,
      driver: this.args?.driver || process.env.MY_DRIVER || 'default'
    };

    const result = await service?.execute(options);

    console.log(JSON.stringify(result, null, 2));
  }
}
```

Key patterns:
- Action methods are named after the method portion of `--action=alias:method`.
  `--action=my-module:execute` dispatches to `MyCLIController.execute()`.
- Read arguments from `this.args?.paramName` (CLI flags) or `process.env.ENV_VAR` (env vars).
  Always provide a fallback chain: CLI arg → env var → hardcoded default.
- Generate a flow ID with `this.getId(this.args as any)` and carry it through all log calls.
- Resolve services on demand inside action methods; do not resolve in the constructor.

---

## 8. Implementing an MCP Controller (src/controllers/MyMCPController.ts)

MCP controllers register tools with the MCP server. The `register(server)` method is called
once during startup; tool handlers are called by the LLM client.

```typescript
import { KzController, VCategory } from '@kozen/engine';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';
import { IMyOptions } from '../models/MyModel';

export class MyMCPController extends KzController {

  public async register(server: McpServer): Promise<void> {
    server.registerTool(
      'kozen_my_execute',
      {
        description: 'Execute the main operation of my-module.',
        inputSchema: {
          key:    z.string().describe('The target key to operate on.'),
          driver: z.string().optional().describe('Backend driver. Defaults to "default".')
        }
      },
      this.execute.bind(this)
    );
  }

  private async execute(args: { key: string; driver?: string }): Promise<{ content: any[] }> {
    const flow = this.getId();

    try {
      const service = await this.assistant?.resolve<
        import('../services/MyService').MyService
      >('my-module:service');

      const options: IMyOptions = {
        key:    args.key,
        driver: args.driver || process.env.MY_DRIVER || 'default'
      };

      const result = await service?.execute(options);

      return {
        content: [{
          type: 'text',
          text: JSON.stringify(result, null, 2)
        }]
      };
    } catch (error) {
      return {
        content: [{
          type: 'text',
          text: `Error: ${(error as Error).message}`
        }]
      };
    }
  }
}
```

Key patterns:
- Tool names follow the convention `kozen_modulealias_action` to avoid collisions across
  modules loaded simultaneously (e.g., `kozen_secret_select`, `kozen_trigger_start`).
- All `inputSchema` fields use Zod schemas. The `.describe()` call provides the field
  description shown to the LLM.
- Return value is always `{ content: Array<{ type: string; text: string }> }`.
- On error, return an error message in the content array rather than throwing — the MCP
  protocol expects a content response, not an exception.

---

## 9. Models file (src/models/MyModel.ts)

Define all public-facing interfaces in a models file. This is what consumers import when
using your module as a library.

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

---

## 10. Building and testing locally

```bash
# Install dependencies
npm install

# Build
npm run build           # tsc + copy non-TS assets

# Test as CLI (via ts-node, no build required)
npm run dev -- --action=my-module:help
npm run dev -- --action=my-module:execute --key=test-key

# Test as MCP (opens MCP Inspector UI at localhost:5173)
npm run mcp:dev

# Link locally to test integration in another project
npm link                                  # creates a global symlink to this package
cd /path/to/host-project
npm link @scope/my-module                 # use the symlinked version
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

---

## 11. Publishing to npm

```bash
# Build production dist
npm run build

# Verify what will be published
npm pack --dry-run

# First publish (scoped public package)
npm publish --access public

# Subsequent publishes: bump version first
npm version patch   # 1.0.0 → 1.0.1
npm publish         # access is remembered after first publish
```

The `files` field in `package.json` already limits the tarball to `dist/`, `README.md`, and
`LICENSE`. Verify the tarball never includes `src/`, `node_modules/`, or `cfg/*.env*`.

Using the module after publishing:

```bash
npm install @scope/my-module
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

---

## 12. Complete minimal module example

This is the smallest valid Kozen module — one service, one CLI controller, no MCP:

**Directory:**
```
@scope/greeter/
├── src/
│   ├── index.ts
│   ├── controllers/GreeterCLIController.ts
│   ├── services/GreeterService.ts
│   └── configs/
│       ├── ioc.json
│       └── cli.json
├── package.json
└── tsconfig.json
```

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
  "greeter:cli": {
    "target": "GreeterCLIController",
    "type": "class",
    "lifetime": "singleton",
    "path": "../controllers",
    "dependencies": [
      { "key": "assistant", "target": "IoC",            "type": "ref" },
      { "key": "logger",    "target": "logger:service", "type": "ref" }
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
  public async greet(): Promise<void> {
    const name = this.args?.name || 'World';
    const svc = await this.assistant?.resolve<GreeterService>('greeter:service');
    const msg = await svc?.greet(name);
    console.log(msg);
  }
}
```

**src/index.ts:**
```typescript
import { IConfig, IDependency, KzModule } from '@kozen/engine';
import cli from './configs/cli.json';
import ioc from './configs/ioc.json';
import fs from 'fs';
import path from 'path';

export class GreeterModule extends KzModule {
  constructor(dependency?: any) {
    super(dependency);
    this.metadata.alias = 'greeter';
    try {
      const pac = JSON.parse(fs.readFileSync(path.resolve(__dirname, '../package.json'), 'utf-8'));
      this.metadata.name    = pac.name;
      this.metadata.version = pac.version;
      this.metadata.uri     = pac.homepage;
    } catch (_) {}
  }

  public register(config: IConfig | null): Promise<Record<string, IDependency> | null> {
    let dep: Record<string, any> = config?.type === 'cli' ? { ...ioc, ...cli } : ioc;
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
