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

## 2. Design principles

A Kozen module is production software. The quality of a module is measured by three things:

1. **How quickly a new developer understands the source code** — clarity over cleverness.
2. **How easily it can be modified or extended** — a change in one place should not break
   another.
3. **How efficiently it uses hardware** — minimum RAM, CPU, and disk I/O to accomplish the
   task. Do not allocate what you do not need; do not block when you can yield.

Apply the principles below as needed. Do not apply them all to every module — apply the ones
that solve a real problem in the code at hand.

---

### KISS — Keep It Simple

The simplest solution that works correctly is the best solution. Complexity introduced
"just in case" or "for future flexibility" is waste. If a feature is not required now, do
not build the infrastructure for it.

- A service with one method is better than a service with ten.
- A flat `switch` is better than a class hierarchy when there are fewer than four cases.
- A plain function is better than a class when there is no state to encapsulate.

---

### SoC — Separation of Concerns

Each class has exactly one reason to change:

- **Services** own business logic. They do not parse CLI flags, format output, or manage
  connections directly — they delegate that to the layer that owns it.
- **Controllers** own I/O dispatch (parsing args, formatting output, routing to services).
  They do not contain business logic.
- **Models** own type definitions. They do not contain methods.

When a class is hard to name, it is doing too many things.

---

### Class hierarchies — avoid the yo-yo problem

Inheritance hierarchies deeper than two levels (base + one concrete subclass) are forbidden.
The yo-yo problem occurs when understanding a method requires jumping up and down a deep
inheritance chain — every jump is a cognitive cost and a maintenance hazard.

```
// ✅ Acceptable: two levels
BaseService → MyService

// ❌ Forbidden: three or more levels
BaseService → AbstractMyService → ConcreteMyService → SpecializedMyService
```

When a third level is tempting, use **composition** instead: extract the varying behavior
into a collaborator and inject it. This is also the foundation of the Strategy pattern
(see below).

---

### SOLID in practice

Apply these four rules; the fifth (Interface Segregation) is usually automatic in TypeScript:

| Principle | Rule in Kozen modules |
|---|---|
| **SRP** — Single Responsibility | One class, one reason to change |
| **OCP** — Open / Closed | Extend by adding a new strategy class, not by modifying existing logic |
| **LSP** — Liskov Substitution | A subclass must be usable wherever its parent is expected — if it isn't, it should not be a subclass |
| **DIP** — Dependency Inversion | Depend on tokens resolved from the IoC container, not on concrete `new MyClass()` calls |

---

### Strategy pattern for variable behavior

Whenever behavior can vary — now or in a likely future — model it as interchangeable
implementations of a shared contract. Classes do not need to be named `*Strategy`; the
concept is what matters: each concrete class defines a distinct behavior, and the caller
holds a reference to the interface, not the implementation.

Real Kozen examples:
- `SecretManagerAWS` and `SecretManagerMDB` are both implementations of `ISecretManager` —
  the `SecretManager` service selects the right one based on config.
- `IAMRectificationScram` and `IAMRectificationX509` implement the same rectification
  contract with different authentication mechanisms.

**Rule:** if the same action can be performed in more than one way, and the choice depends
on configuration, create one class per variant and select via IoC or a factory — do not add
`if/switch` logic inside a single class that grows with every new variant.

---

### Utility functions vs services

Before writing a utility function, ask: does this logic depend on injected state (a
connection, a config, a logger)? If yes, it is a service class that must be registered in
`ioc.json` and resolved through IoC. If no, a plain exported function is correct.

| Situation | Solution |
|---|---|
| Stateless transformation (parse, format, convert) | Plain exported function in `src/utils/` |
| Logic that needs a DB connection, logger, or config | `BaseService` subclass in `ioc.json` |
| Logic shared across multiple services in the same module | Promote to a service registered in `ioc.json` |
| Logic shared across multiple Kozen modules | Publish as a separate npm package (see `@mongodb-solution-assurance/iam-util`) |

Keep utility functions minimal. If you find yourself building a utility library inside a
module, extract it into its own package or reconsider whether the module is focused enough.

---

### IoC as the single dynamic loader

**All dynamic class instantiation and resource loading must go through the IoC container.**
Never use `new MyService()` inside a controller or service body. Never use `require()` or
`import()` to load a class at runtime outside of the IoC registration phase.

```typescript
// ✅ Correct: resolve through IoC
const service = await this.assistant?.resolve<IMyService>('my-module:service');

// ❌ Wrong: direct instantiation bypasses the container
const service = new MyService(this.assistant, this.logger);
```

Reasons:
- The IoC container manages lifetime (singleton, transient, scoped) — bypassing it creates
  uncontrolled duplicate instances.
- Resolving through a token string means the implementation can be swapped without changing
  the caller (DIP + Strategy in one move).
- It makes dependencies explicit and testable.

---

### Circular imports — always forbidden

A circular import occurs when module A imports from module B and module B imports from module
A (directly or transitively). In Node.js/TypeScript the runtime resolves one of the modules
first and hands the other an empty or partial object, causing `undefined` errors that appear
only at call time and are difficult to trace.

**Enforce a strict one-way import direction across all source layers:**

```
src/models/      ← no imports from other src/ layers
    ↑
src/services/    ← imports types from src/models/ only
    ↑
src/controllers/ ← imports types from src/models/; resolves services via IoC token
    ↑
src/index.ts     ← imports and re-exports public types; registers the module
```

Rules that follow from this direction:

- **`src/models/`** contains only interfaces and type aliases. It never imports from
  `src/services/` or `src/controllers/`.
- **`src/services/`** imports types from `src/models/` for its signatures. It never imports
  another service class directly — cross-service dependencies are always resolved through the
  IoC container by token string.
- **`src/controllers/`** imports types from `src/models/` for parameter and return types. It
  never imports a service class — services are resolved through IoC.
- **`src/index.ts`** is the only file allowed to import from all layers; it exports the
  public API and registers the module. Nothing else imports from `src/index.ts`.

```typescript
// ✅ Correct: controller resolves service through IoC — no import of the class
import type { IMyService } from '../models/MyModel';
const svc = await this.assistant?.resolve<IMyService>('my-module:service');

// ❌ Wrong: direct import creates a dependency edge that can produce a cycle
import { MyService } from '../services/MyService';
const svc = new MyService(this.assistant, this.logger);
```

If a lint tool or TypeScript path analysis reports a cycle, treat it as a build error, not a
warning. The fix is always to break the dependency by moving shared types to `src/models/`
and resolving the runtime dependency through IoC.

---

### GRASP quick reference

Apply these GRASP responsibilities as a checklist when assigning behavior to a class:

| Responsibility | Question | Kozen application |
|---|---|---|
| **Information Expert** | Who has the data needed to fulfil this? | The service that owns the domain data handles the operation — not the controller |
| **Creator** | Who should create instance X? | The IoC container — not a controller or another service |
| **Low Coupling** | Does this dependency need to exist? | Inject via IoC token, not via `new` — replace the dependency without touching the consumer |
| **High Cohesion** | Do all methods in this class belong together? | If a class has methods that rarely interact with each other, split it |
| **Controller** | Who receives the system event? | `KzController` subclass receives the CLI/MCP event and delegates to services |

---

### GoF patterns in use

| Pattern | Where it appears in Kozen |
|---|---|
| **Strategy** | `ISecretManager` → `SecretManagerAWS` / `SecretManagerMDB`; `IAMRectification*` variants |
| **Template Method** | `KzModule.register()` defines the skeleton; subclasses fill in the dependency map |
| **Facade** | `KzController` subclasses expose a simple action interface over one or more services |
| **Singleton** | IoC `lifetime: "singleton"` — prefer this for stateless services; never implement Singleton manually |
| **Factory Method** | The IoC container is the factory — register a `function` type dependency when construction logic is needed |

---

### Async by default — never block the event loop

Node.js runs on a single-threaded event loop. Any synchronous blocking call — file reads,
DNS lookups, cryptographic operations — freezes the loop for every other concurrent
operation (other requests, log flushes, timers). In a Kozen module running as an MCP server
or long-lived daemon this is unacceptable.

**Rule: use the async variant of every I/O API. The sync variant is forbidden unless the
call is provably unreachable during normal operation and the performance impact is justified
and documented.**

```typescript
// ✅ Correct — yields to the event loop while waiting for the file
import { readFile } from 'fs/promises';
const content = await readFile(path, 'utf-8');

// ❌ Wrong — blocks the entire process until the file is read
import { readFileSync } from 'fs';
const content = readFileSync(path, 'utf-8');
```

The rule applies to every I/O boundary:

| Domain | Use | Avoid |
|---|---|---|
| File system | `fs/promises` (`readFile`, `writeFile`, `mkdir`, …) | `fs.readFileSync`, `fs.writeFileSync`, … |
| MongoDB driver | `await collection.findOne()`, `await cursor.toArray()` | synchronous cursor iteration |
| Network / HTTP | `fetch`, `axios` with `await` | any blocking HTTP client |
| Crypto | `crypto.subtle.*`, `bcrypt.hash()` | `crypto.hashSync`, `bcryptSync` |
| Child processes | `execFile` / `spawn` with `await` | `execFileSync`, `spawnSync` |
| `setTimeout` delays | `await new Promise(r => setTimeout(r, ms))` | `Atomics.wait()`, busy-wait loops |

**The one narrow exception:** a synchronous call is acceptable only when all of these
conditions are true simultaneously:

1. It runs exclusively at **module startup** (inside `register()` or a constructor), before
   any concurrent I/O has started.
2. The data read is **small and bounded** (a local config file, a single JSON file).
3. The sync call executes **at most once** per process lifetime.
4. A JSDoc comment above the call states **why** the sync variant was chosen.

If any condition is not met, use the async variant.

---

## 3. package.json

The template below matches the patterns used across `@kozen/trigger`, `@kozen/secret`, and
`@kozen/iam-rectification`. Annotations in comments explain every field and flag.

```json
{
  "name": "@scope/my-module",
  "version": "1.0.0",
  "description": "Kozen module that provides [domain] capabilities",

  "main":  "dist/index.js",
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
    "build":        "tsc && npm run copy:txt",
    "copy:txt":     "copyfiles -u 1 src/**/*.txt dist/",
    "test":         "echo \"Error: no test specified\" && exit 1",
    "start":        "npm run bin:kozen:js -- --config=kozen.js.json",
    "dev":          "npm run bin:kozen:ts -- --config=kozen.ts.json",
    "mcp:start":    "npm run bin:kozen:ts -- --type=mcp",
    "mcp:dev":      "npx -y @modelcontextprotocol/inspector npm run bin:kozen:ts -- --type=mcp",
    "bin:kozen:ts": "ts-node node_modules/@kozen/engine/dist/bin/kozen.js",
    "bin:kozen:js": "npx kozen",
    "versions":     "npm view @scope/my-module versions",
    "publish":      "npm run build && npm publish",
    "clean":        "rm -rf dist"
  },

  "keywords": ["kozen", "mongodb", "my-domain"],
  "author":   "Your Name or Organisation",
  "license":  "MIT",

  "engines": { "node": ">=18" },

  "homepage": "https://github.com/your-org/my-module/wiki",
  "bugs":     { "url": "https://github.com/your-org/my-module/issues" },
  "repository": {
    "type": "git",
    "url":  "git+https://github.com/your-org/my-module.git"
  },

  "dependencies": {
    "@kozen/engine": "^1.1.15"
  },

  "devDependencies": {
    "@types/node": "^18.19.0",
    "copyfiles":   "^2.4.1",
    "cross-env":   "^7.0.3",
    "ts-node":     "^10.9.0",
    "tslint":      "^5.8.0",
    "typescript":  "^5.0.0"
  }
}
```

### Key rules

| Field / script | Rule |
|---|---|
| `main` / `types` | Must point to `dist/index.js` / `dist/index.d.ts` (see rootDir rule in section 3) |
| `files` | `"dist/*"` publishes only compiled output — `src/`, `cfg/`, and `.env` files are excluded |
| `publishConfig.access` | `"public"` is required for all `@scope/*` packages; without it npm defaults to private |
| `@kozen/engine` in `dependencies` | Must be a runtime `dependency`, not `devDependency`; it is imported at module execution time |
| `zod` | Add to `dependencies` only if the module defines MCP tools (Zod schemas for `inputSchema`); omit otherwise |
| `cross-env` | Add to `devDependencies` when scripts need cross-platform environment variable assignment |
| `tslint` | Present in all existing Kozen modules (legacy linter); add for consistency. `eslint` is the modern replacement — use either, but be consistent within the project |
| `@types/node` version | Pin to the same major as `engines.node`: `node >=18` → `@types/node ^18.x.x` |
| `engines.node` | Set to `">=18"` — the minimum Node.js version Kozen supports |
| `test` script | Always include; use the placeholder if there are no tests yet — `npm test` must not crash |
| `clean` script | `rm -rf dist` — useful for CI and before a fresh build |

### Config file naming convention

Kozen modules conventionally ship two config files at the project root for local development:

| File | Used by | Purpose |
|---|---|---|
| `kozen.js.json` | `npm start` (compiled run) | Config for running the module with the published `npx kozen` binary |
| `kozen.ts.json` | `npm run dev` (ts-node run) | Config for running during development via `ts-node` |

Both files are kozen config files (same schema as `cfg/config.json`). The split exists because
`module` path resolution differs between the compiled `dist/` and the source `src/` trees during
development. Neither file should be committed with credentials.

The `start` and `dev` scripts pass the config file via `-- --config=<file>`:

```bash
# Pass args to a script via npm (the '--' separator is required)
npm run start               # → npx kozen --config=kozen.js.json
npm run dev                 # → ts-node ...kozen.js --config=kozen.ts.json
npm run dev -- --action=my-module:help   # add extra flags at call time
```

When no config file is needed (the action is specified entirely via CLI flags), use:

```bash
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

### Entry point paths and the rootDir inference rule

The `main` and `types` fields must match the compiled output path. That path is controlled by
how TypeScript infers `rootDir` from the `include` patterns:

| `include` in tsconfig | Inferred `rootDir` | `src/index.ts` compiles to | `main` field |
|---|---|---|---|
| `["src/**/*.ts"]` — all modules | `src/` | `dist/index.js` | `"dist/index.js"` |
| `["src/**/*.ts", "bin/**/*.ts"]` — engine only | `.` (project root) | `dist/src/index.js` | `"dist/src/index.js"` |

**Rule for modules:** only include `src/**/*.ts`. Never add `bin/**/*.ts` to a module's
tsconfig — Kozen modules do not define their own CLI binary. Adding `bin/**/*.ts` shifts the
output path to `dist/src/index.js` and breaks the `main` field.

This is also why the `copy:txt` script differs between the engine and every other module:

| Package | `copy:txt` script | Why |
|---|---|---|
| `@kozen/engine` | `copyfiles -u 1 src/**/*.txt dist/src` | rootDir is `.`, docs land in `dist/src/docs/` |
| All modules | `copyfiles -u 1 src/**/*.txt dist/` | rootDir is `src/`, docs land in `dist/docs/` |

### npm scripts reference

All scripts are part of the template above. Reference table:

| Script | Command | Purpose |
|---|---|---|
| `build` | `tsc && npm run copy:txt` | TypeScript compilation + copy `.txt` docs to `dist/` |
| `copy:txt` | `copyfiles -u 1 src/**/*.txt dist/` | Copy help docs; also called by `build` |
| `test` | `echo "Error: no test specified" && exit 1` | Placeholder until tests exist; prevents `npm test` from silently succeeding |
| `start` | `npm run bin:kozen:js -- --config=kozen.js.json` | Run the compiled module with the published `kozen` binary |
| `dev` | `npm run bin:kozen:ts -- --config=kozen.ts.json` | Run via `ts-node` — no build step needed |
| `mcp:start` | `npm run bin:kozen:ts -- --type=mcp` | Start the MCP server via `ts-node` |
| `mcp:dev` | `npx -y @modelcontextprotocol/inspector npm run bin:kozen:ts -- --type=mcp` | Open MCP Inspector UI at `localhost:5173` for debugging |
| `bin:kozen:ts` | `ts-node node_modules/@kozen/engine/dist/bin/kozen.js` | Engine bin via `ts-node` — base script for `dev` and `mcp:start` |
| `bin:kozen:js` | `npx kozen` | Published `kozen` binary — base script for `start` |
| `versions` | `npm view @scope/my-module versions` | List all published versions — run before bumping |
| `publish` | `npm run build && npm publish` | Build then publish in one step |
| `clean` | `rm -rf dist` | Wipe compiled output before a clean build |

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

## 4. tsconfig.json

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2020",
    "module": "CommonJS",
    "lib": ["ES2020"],
    "outDir": "dist",
    "moduleResolution": "node",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "pretty": true,
    "typeRoots": ["./node_modules/@types"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

Key compiler options for Kozen modules:

| Option | Value | Why |
|---|---|---|
| `module` | `"CommonJS"` | All Kozen modules publish CJS for Node.js compatibility |
| `target` | `"ES2020"` | Supports async/await, optional chaining, and modern JS natively |
| `declaration` | `true` | Generates `.d.ts` files consumed by TypeScript projects that import the module |
| `declarationMap` | `true` | Maps `.d.ts` declarations back to source — enables "Go to definition" in IDEs |
| `resolveJsonModule` | `true` | Required for `import ioc from './configs/ioc.json'` in `src/index.ts` |
| `skipLibCheck` | `true` | Prevents type errors in `node_modules/@types` from blocking your build |
| `experimentalDecorators` | `true` | Required if any service uses class decorators (some IoC patterns use them) |
| `emitDecoratorMetadata` | `true` | Required alongside `experimentalDecorators` for runtime reflection |
| `esModuleInterop` | `true` | Enables `import dotenv from 'dotenv'` (default import from CJS modules) |
| `allowSyntheticDefaultImports` | `true` | Companion to `esModuleInterop`; silences type errors on CJS default imports |
| `forceConsistentCasingInFileNames` | `true` | Prevents case-sensitivity bugs when developing on macOS/Windows and deploying on Linux |
| `noImplicitReturns` | `true` | Catches missing `return` statements in async methods |
| `paths` | `{ "@/*": ["./src/*"] }` | Enables `@/services/MyService` import aliases within the module source |

**Critical: `include` must be `["src/**/*.ts"]` only.**
Do not add `bin/**/*.ts`. Modules have no `bin/` directory. Including it would shift the
inferred `rootDir` from `src/` to `.`, causing `src/index.ts` to compile to `dist/src/index.js`
instead of `dist/index.js` — breaking the `main` field in `package.json`.

---

## 5. The module entry point (src/index.ts)

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

## 6. The `register()` function in depth

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

## 7. IoC configuration files — why three files, not one

Most developers ask: "Why not put everything in one `ioc.json`?" The answer is
**conditional loading by runtime type**.

### How dependency injection works — the key-to-parameter mapping

Every entry in the `dependencies` array has a `"key"` field. That key becomes the **property
name** on the `dependency` object the IoC container builds and passes to the class constructor.
The TypeScript constructor must declare a matching parameter name to receive the injected
instance.

```
JSON config                                TypeScript constructor
─────────────────────────────────          ─────────────────────────────────────
"key": "assistant" → target: "IoC"    →   dependency?.assistant  (IIoC)
"key": "logger"    → target: "logger:service" → dependency?.logger (ILogger)
"key": "srvFile"   → target: "core:file" →   dependency?.srvFile  (FileService)
"key": "srvIAM"    → target: "iam:service" →  dependency?.srvIAM   (IIAMService)
```

**Concrete example — the same registration shown as JSON and TypeScript side-by-side:**

```json
{
  "iam-rectification:controller:mcp": {
    "target": "RectificationMCPController",
    "type": "class",
    "lifetime": "transient",
    "path": "controllers",
    "dependencies": [
      { "key": "assistant", "target": "IoC",              "type": "ref" },
      { "key": "logger",    "target": "logger:service",   "type": "ref" },
      { "key": "srvFile",   "target": "core:file",        "type": "ref" },
      { "key": "srvIAMScram","target": "iam:service:scram","type": "ref" },
      { "key": "srvIAMX509","target": "iam:service:x509", "type": "ref" }
    ]
  }
}
```

```typescript
export class RectificationMCPController extends MCPController {

  private srvFile?: FileService;
  private srvIAMScram?: IIAMRectification;
  private srvIAMX509?: IIAMRectification;

  constructor(dependency?: {
    assistant:  IIoC;
    logger:     ILogger;
    srvFile?:   FileService;
    srvIAMScram?: IIAMRectification;
    srvIAMX509?:  IIAMRectification;
  }) {
    super(dependency);                   // passes assistant + logger to KzController
    this.srvFile     = dependency?.srvFile;
    this.srvIAMScram = dependency?.srvIAMScram;
    this.srvIAMX509  = dependency?.srvIAMX509;
  }
}
```

Key rules:
- **`"target": "IoC"`** is the only reserved special token — it injects the container itself.
- **`"target": "logger:service"`** resolves the engine's built-in `LoggerService` singleton.
- **`"target": "core:file"`** resolves the engine's built-in `FileService` (CSV/JSON I/O).
- Any other `"target"` value must be a token already registered in `ioc.json` (or the
  merged overlay file) before Awilix resolves the current entry.
- **`"type": "ref"`** is the only dependency type used in practice — it resolves by token name.
- The constructor `dependency` object is always optional (`dependency?`) because the class
  may also be instantiated directly in unit tests without an IoC container.



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

## 8. Implementing a Service (src/services/MyService.ts)

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

## 9. Implementing a CLI Controller (src/controllers/MyCLIController.ts)

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
      key:    args.key    || process.env.KOZEN_MY_MODULE_KEY    || '',
      driver: args.driver || process.env.KOZEN_MY_MODULE_DRIVER || 'default'
    };

    const result = await this.srvMyModule?.execute(options);
    console.log(JSON.stringify(result, null, 2));
  }

  /** Apply env-var fallbacks and default values to parsed args. */
  public async fill(args: string[] | IArgs): Promise<IArgs> {
    const parsed = await super.fill(args);
    parsed['key']    = parsed['key']    || process.env.KOZEN_MY_MODULE_KEY;
    parsed['driver'] = parsed['driver'] || process.env.KOZEN_MY_MODULE_DRIVER || 'default';
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
- **Env var naming: `KOZEN_<MODULE_ALIAS>_<PARAM>`.** See the naming convention below.

### Environment variable naming convention

All environment variables introduced by a Kozen module must follow this pattern:

```
KOZEN_<MODULE_ALIAS>_<PARAMETER>
```

Where `MODULE_ALIAS` is the value of `this.metadata.alias`, uppercased, with hyphens
replaced by underscores.

| Module alias | Prefix | Example variable |
|---|---|---|
| `trigger` | `KOZEN_TRIGGER_` | `KOZEN_TRIGGER_URI`, `KOZEN_TRIGGER_DB` |
| `iam-rectification` | `KOZEN_IAM_RECTIFICATION_` | `KOZEN_IAM_RECTIFICATION_URI` |
| `my-module` | `KOZEN_MY_MODULE_` | `KOZEN_MY_MODULE_KEY`, `KOZEN_MY_MODULE_DRIVER` |

**Exception — reuse existing well-known variables.** When the parameter maps directly to an
established ecosystem convention, use that convention instead of inventing a new prefixed name:

| Existing variable | Use instead of | Context |
|---|---|---|
| `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | `KOZEN_SECRET_AWS_REGION` etc. | AWS SDK reads these natively |
| `MDB_MASTER_KEY` | `KOZEN_SECRET_MASTER_KEY` | Established MongoDB CSFLE convention across the ecosystem |
| `MDB_URI`, `MONGODB_URI` | `KOZEN_MY_MODULE_URI` | Only when the variable is already present in the project's `.env` by convention |

The rule of thumb: if the variable would already exist in the user's environment from another
tool or SDK, reuse it. If it is new and Kozen-specific, prefix it.

**Apply the convention consistently in three places:**
1. `fill()` in the CLI controller — `process.env.KOZEN_MY_MODULE_PARAM`
2. `src/docs/<alias>.txt` — list the variable in the `Environment Variables` section
3. The module's wiki `Configuration.md` page — document it in the env vars table

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

## 10. Implementing an MCP Controller (src/controllers/MyMCPController.ts)

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
        driver: args.driver || process.env.KOZEN_MY_MODULE_DRIVER || 'default'
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

## 11. Models file (src/models/MyModel.ts)

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

## 12. Structured logging

Every Kozen module uses the `LoggerService` that ships with `@kozen/engine`. It is registered
as a singleton in the IoC container under the token `logger:service` and injected into every
service and controller that declares the dependency. **Never use `console.log` directly** —
all output must go through `this.logger` so that level filtering, flow correlation, and
multi-destination routing (console / MongoDB / file) work consistently.

For the full programmatic API, environment variables, and output destination configuration,
see `references/cli-mcp.md`. This section covers only what is needed while writing module
source code.

---

### Injecting the logger via IoC

`logger:service` is a pre-registered singleton in `@kozen/engine`. Every service or
controller that wants to log simply declares `"key": "logger"` in its `dependencies` array
and receives a `LoggerService` instance as `dependency.logger` in the constructor.

**Service — ioc.json entry and matching TypeScript constructor:**

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

```typescript
export class MyService extends BaseService {
  constructor(dependency?: { assistant: IIoC; logger: ILogger }) {
    super(dependency); // BaseService stores dependency.assistant → this.assistant
                       //                    dependency.logger    → this.logger
  }
}
```

**Controller — cli.json entry with assistant, logger, and additional services:**

```json
{
  "my-module:controller:cli": {
    "target": "MyCLIController",
    "type": "class",
    "lifetime": "transient",
    "path": "controllers",
    "dependencies": [
      { "key": "assistant",  "target": "IoC",              "type": "ref" },
      { "key": "logger",     "target": "logger:service",   "type": "ref" },
      { "key": "srvFile",    "target": "core:file",        "type": "ref" },
      { "key": "srvMyModule","target": "my-module:service","type": "ref" }
    ]
  }
}
```

```typescript
export class MyCLIController extends CLIController {

  private srvMyModule?: IMyService;

  constructor(dependency?: {
    assistant:    IIoC;
    logger:       ILogger;
    srvFile?:     FileService;  // engine built-in — CSV/JSON I/O utility
    srvMyModule?: IMyService;   // registered in ioc.json of this module
  }) {
    super(dependency);                          // wires assistant + logger + srvFile
    this.srvMyModule = dependency?.srvMyModule; // additional services stored manually
  }
}
```

The JSON `"key"` value is **always identical** to the TypeScript constructor parameter name.
`super(dependency)` (inherited from `KzController`) stores `assistant`, `logger`, and
`srvFile` automatically — any extra service key must be stored manually in the constructor
body.

---

### ILogLevel values

```typescript
import { ILogLevel } from '@kozen/engine';

ILogLevel.NONE    // 0  — no output (required for MCP servers)
ILogLevel.ERROR   // 1  — errors only
ILogLevel.WARN    // 2  — errors + warnings
ILogLevel.DEBUG   // 3  — errors + warnings + debug
ILogLevel.INFO    // 4  — all standard levels (default)
ILogLevel.ALL     // -1 — everything, including verbose
```

---

### ILogEntry — the full structure

Every `this.logger.*()` call accepts either a plain string or an `ILogEntry` object:

```typescript
interface ILogEntry {
  message:   string;          // required — human-readable description
  flow?:     string;          // correlation ID (YYYYMMDDHHMMSSXX); auto-generated if omitted
  src?:      string;          // 'alias:layer:method', e.g. 'trigger:service:start'
  category?: string;          // VCategory.* constant or a plain string tag
  data?:     any;             // structured payload (object, array, primitive)
  level?:    string | ILogLevel; // rarely set manually; the method name sets this
  date?:     string;          // ISO timestamp; auto-generated if omitted
}
```

---

### Flow IDs — generation and propagation

A **flow ID** links every log entry for a single end-to-end operation (one CLI invocation,
one MCP tool call, one SDK call). The format is `YYYYMMDDHHMMSSXX` where `XX` is a random
two-digit suffix.

**In a controller** — generate once at the start of each action and pass it down:

```typescript
public async start(options: IArgs): Promise<void> {
  const flow = this.getId(options as unknown as IConfig); // reads --flow or auto-generates
  this.logger?.info({ flow, src: 'trigger:controller:start', message: 'Action started' });
  const svc = await this.assistant?.resolve<IChangeStreamService>('trigger:service');
  await svc?.start({ ...options, flow });
}
```

**In a service** — receive `flow` through the options object and thread it through every log
call within that invocation:

```typescript
public async start(options: ITriggerOptions): Promise<void> {
  const { flow } = options;
  this.logger?.info({
    flow,
    src: 'trigger:service:start',
    message: 'Change-stream watcher started',
    data: { database: options.mdb.database, collection: options.mdb.collection }
  });
  // ... on error:
  this.logger?.error({
    flow,
    src: 'trigger:service:start',
    message: `Failed to open change stream: ${(error as Error).message}`
  });
}
```

Never generate a new flow ID inside a service — only controllers or the top-level entry
point create flow IDs.

---

### `src` naming convention

Use the pattern `'alias:layer:method'`, all lowercase, colon-separated:

| Example | Where |
|---|---|
| `'trigger:service:start'` | `ChangeStreamService.start()` |
| `'trigger:controller:start'` | `TriggerCLIController.start()` |
| `'secret:service:manager:get'` | `SecretManager.get()` inside the secret module |
| `'iam-rectification:controller:verify'` | `RectificationCLIController.verify()` |

---

### Logger in a service (complete example)

```typescript
import { BaseService, ILogLevel } from '@kozen/engine';
import type { IIoC, ILogger } from '@kozen/engine';
import type { IMyOptions } from '../models/MyModel';

export class MyService extends BaseService {

  constructor(dependency?: { assistant: IIoC; logger: ILogger }) {
    super(dependency);
  }

  /**
   * Execute the main operation.
   * @param {IMyOptions} options - Runtime options including the flow ID.
   */
  public async execute(options: IMyOptions): Promise<void> {
    const { flow } = options;
    try {
      this.logger?.debug({ flow, src: 'my-module:service:execute', message: 'Starting' });
      // ... business logic ...
      this.logger?.info({ flow, src: 'my-module:service:execute', message: 'Completed' });
    } catch (error) {
      this.logger?.error({
        flow,
        src: 'my-module:service:execute',
        message: (error as Error).message
      });
      throw error;
    }
  }
}
```

---

### Logger in a controller — `getId()` and `wait()`

```typescript
import { CLIController } from '@kozen/engine';
import type { IIoC, ILogger, IArgs, IConfig, FileService } from '@kozen/engine';
import type { IMyService } from '../models/MyModel';

export class MyCLIController extends CLIController {

  private srvMyModule?: IMyService;

  // The constructor parameter names must match the "key" values in cli.json exactly.
  constructor(dependency?: {
    assistant:    IIoC;
    logger:       ILogger;
    srvFile?:     FileService;  // stored automatically by super()
    srvMyModule?: IMyService;   // must be stored manually
  }) {
    super(dependency);                          // stores assistant, logger, srvFile
    this.srvMyModule = dependency?.srvMyModule;
  }

  /**
   * Execute the action. Flow ID is generated once here and threaded to services.
   */
  public async execute(options: IArgs): Promise<void> {
    // getId() reads --flow from options or generates a YYYYMMDDHHMMSSXX timestamp
    const flow = this.getId(options as unknown as IConfig);
    try {
      this.logger?.info({ flow, src: 'my-module:controller:execute', message: 'Action started' });
      await this.srvMyModule?.execute({ ...options, flow });
      this.logger?.info({ flow, src: 'my-module:controller:execute', message: '✅ Done' });
    } catch (error) {
      this.logger?.error({
        flow,
        src: 'my-module:controller:execute',
        message: `❌ ${(error as Error).message}`
      });
    }
    // Flush all async log writes before the process exits
    await this.wait();
  }
}
```

`this.wait()` (inherited from `KzController`) calls `await Promise.all(this.logger.stack)`.
Always call it at the end of every action that logs — especially before process exit.

`this.srvMyModule` is used directly (injected via IoC, not resolved at call time) because
the controller holds a reference set during construction. For services resolved lazily
(only when needed), use `await this.assistant?.resolve<IMyService>('my-module:service')`
inside the action method instead.

---

### VCategory constants

`VCategory` is exported by `@kozen/engine` and provides standard category strings for
consistent log filtering. Prefer these over ad-hoc strings:

```typescript
import { VCategory } from '@kozen/engine';

VCategory.core.secret    // 'CORE.SECRET'
VCategory.core.trigger   // 'CORE.TRIGGER'
// Use VCategory.core.<alias> in the category field when the module alias has a match.
// For custom modules, use a plain UPPER_SNAKE string: 'MY_MODULE'.
```

---

## 13. Code comment standard (JSDoc)

All Kozen module source code uses JSDoc syntax (`/** ... */`) for API-level comments. Follow
the standard multi-line JSDoc format described at https://jsdoc.app/howto-es2015-classes.
The rules below apply to every file in `src/`.

### Rules

- **Always use the multi-line JSDoc block format** — even for a single sentence. Never
  collapse a comment to `/** text */` on one line; always open with `/**`, prefix each body
  line with ` * `, and close with ` */`.
- **Keep comments brief.** A description should fit in one to two lines. If more detail is
  needed, move it to `src/docs/<alias>.txt`.
- **Document the WHY, not the WHAT.** If the comment restates the method name or its
  signature, delete it. Write a comment only when there is a non-obvious constraint,
  invariant, or side-effect a reader would miss.
- **No `@example` sections.** Examples belong in `src/docs/<alias>.txt` where they are
  versioned and surfaced by the `help()` action. A JSDoc `@example` in source is duplicated
  effort and tends to go stale.
- **Use `@param` for non-obvious parameters.** Include the type (in curly braces) and a short
  dash-separated description following JSDoc convention: `@param {type} name - Description.`
  Skip it for self-evident parameters like `args`, `options`, or `config` when the type
  annotation already documents them.
- **Use `@returns` when the resolved type is not obvious** from the return type annotation.
- **`@deprecated` and `@throws` are always correct** when applicable — include them.

### What to annotate

| Element | Annotate? |
|---|---|
| Public class | Yes |
| Public method that maps to a CLI action | Yes |
| `help()` method — state the alias of the file it reads | Yes |
| `fill()` override — summarize what defaults it applies | Yes |
| `register()` override | Yes |
| `constructor` — only if it has a non-obvious side-effect | Conditional |
| Private utility method — only if the logic is non-obvious | Conditional |
| Interface field — only if the name is ambiguous (skip `uri`, document `delegate`) | Conditional |

### Correct examples (from real module patterns)

```typescript
/** Manages secrets stored in AWS Secrets Manager or MongoDB CSFLE. */
class SecretService extends BaseService {

    /**
     * Retrieve the secret identified by key, using the configured backend.
     * Backend is selected from config at construction time — AWS or MongoDB.
     */
    public async get(): Promise<void> { ... }

    /**
     * Apply env-var fallbacks for URI, database, and collection.
     * @param {string[] | IArgs} args - Raw CLI arguments or pre-parsed map.
     * @returns {Promise<IArgs>} Arguments with environment defaults applied.
     */
    public async fill(args: string[] | IArgs): Promise<IArgs> { ... }

    /**
     * Prints help from src/docs/secret.txt.
     */
    public async help(): Promise<void> { ... }

    /**
     * Starts the change-stream watcher and dispatches events to the delegate.
     * Must be called after the IoC container is fully resolved.
     */
    public async start(options: ITriggerOptions): Promise<void> { ... }
}
```

### Incorrect examples (violations)

```typescript
// ❌ Collapsed single-line format — use multi-line block instead
/** Retrieve the secret identified by key. */
public async get(): Promise<void> {}

// ❌ Restates the method name — no value added
/**
 * Executes the execution.
 */
public async execute() {}

// ❌ Example in JSDoc — belongs in src/docs/<alias>.txt
/**
 * Get a secret.
 * @example
 * kozen --action=secret:get --key=MY_KEY
 */
public async get() {}

// ❌ @param for obvious arg — type annotation already documents it
/**
 * @param {IArgs} args - Parsed CLI arguments.
 * @returns {Promise<IArgs>} Parsed CLI arguments with fallbacks applied.
 */
public async fill(args: IArgs): Promise<IArgs> {}
```

### Extended documentation goes to `src/docs/<alias>.txt`

When a developer needs usage examples, flag tables, or any content that would span more than
two lines in JSDoc, write it in `src/docs/<alias>.txt`, not in JSDoc. That file is
distributed in the npm package, surfaced by `help()`, and readable by LLMs and documentation
generators without executing the code.

---

## 14. Inline module documentation (src/docs/my-module.txt)

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
  KOZEN_CONFIG                 Path to config.json
  KOZEN_ACTION                 Default action
  KOZEN_STACK                  Environment name
  KOZEN_PROJECT                Project ID
  KOZEN_MY_MODULE_PARAM1       <Description>. Maps to --param1.
  KOZEN_MY_MODULE_PARAM2       <Description>. Maps to --param2.

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
  KOZEN_MY_MODULE_PARAM1=value KOZEN_ACTION=my-module:action1 kozen
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

## 15. Building and testing locally

```bash
# Install dependencies
npm install

# Build: TypeScript compilation + copy .txt help files to dist/
npm run build

# Verify the dist/ structure after build
# Expected for a module named 'trigger' with alias 'trigger':
# dist/
# ├── index.js          ← compiled src/index.ts
# ├── index.d.ts        ← TypeScript declarations
# ├── index.js.map
# ├── controllers/
# ├── services/
# ├── models/
# ├── configs/
# └── docs/
#     └── trigger.txt   ← copied by copy:txt — MUST exist

# Test as CLI during development (ts-node, no build step)
npm run dev -- --moduleLoad=. --action=my-module:help
npm run dev -- --moduleLoad=. --action=my-module:execute --key=myTestKey

# Test MCP (opens MCP Inspector UI at localhost:5173)
npm run mcp:dev

# Test as an installed package in another project
npm link
cd /path/to/host-project
npm link @scope/my-module
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

Checklist after `npm run build`:

- [ ] `dist/index.js` exists (compiled entry point)
- [ ] `dist/index.d.ts` exists (TypeScript declarations)
- [ ] `dist/docs/<alias>.txt` exists (help file copied by `copy:txt`)
- [ ] `dist/` does NOT contain `src/` directory (wrong rootDir — fix tsconfig `include`)
- [ ] `dist/` does NOT contain `node_modules/` or `*.test.js`

---

## 16. The engine binary — why modules do not define their own `bin`

`@kozen/engine` is the only package in the Kozen ecosystem that defines a CLI binary. All
other modules are loaded by that binary at runtime via `--moduleLoad`.

### How the engine binary works

`bin/kozen.ts` (compiled to `dist/bin/kozen.js`) is the process entry point. It:

1. Parses `process.argv` via `KzApp.extract()`.
2. Reads `KOZEN_APP_TYPE` (or `--type`) to determine CLI vs MCP.
3. Loads `.env` via `dotenv` (unless `KOZEN_SKIP_DOTENV=true` or type is `mcp`).
4. Calls `KzApp.init()` to build the config.
5. Calls `KzApp.register()` which loads each module listed in `--moduleLoad`.
6. Resolves the application (`CLIApplication` or `MCPApplication`) from the IoC container.
7. Calls `app.start(args)` which dispatches to the correct controller method.

The `#!/usr/bin/env node` shebang on line 1 of the compiled `.js` file makes the file
directly executable on Unix systems without a `node` prefix.

### Why modules do NOT have their own `bin`

A Kozen module is a **plugin**, not a standalone application. Its entry point is
`src/index.ts` which exports a `KzModule` subclass. The module has no `main()` function —
it only exposes a dependency map that the engine registers in its IoC container.

Defining a `bin` field in a module's `package.json` would:
- Add `bin/**/*.ts` to `include`, shifting the tsconfig rootDir and breaking `dist/index.js`
- Duplicate the engine's bootstrap logic in every module
- Create a separate `npx @scope/my-module` command that would not know how to load other
  modules alongside it

Instead, users always invoke modules through the engine binary:

```bash
# CORRECT: use the engine binary and load the module
npx kozen --moduleLoad=@scope/my-module --action=my-module:execute --key=VALUE

# WRONG: there is no separate binary for modules
npx @scope/my-module --action=execute  # will fail — no bin defined
```

### The `bin` field in `package.json` — engine only

```json
// @kozen/engine package.json — the only package with a bin field
{
  "bin": {
    "kozen": "dist/bin/kozen.js"
  }
}
```

When `@kozen/engine` is installed globally (`npm install -g @kozen/engine`) or as a
dependency, npm symlinks `dist/bin/kozen.js` as the `kozen` executable. This is why
`npx kozen` works in any project that has `@kozen/engine` as a dependency.

**Never add a `bin` field to a Kozen module's `package.json`.**

---

## 17. Publishing to npm

### Prerequisites

```bash
# One-time: authenticate with npm
npm login
# Enter: username, password, email, OTP (if 2FA enabled)

# Verify authentication
npm whoami
# → your-npm-username
```

For CI/CD pipelines, use a publish token in `~/.npmrc` instead of interactive login:

```
# ~/.npmrc  (or .npmrc in the project root — do not commit if it contains a token)
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

Set `NPM_TOKEN` as an environment secret in GitHub Actions / GitLab CI.

### Pre-publish checklist

Run all of these before executing `npm publish`:

```bash
# 1. Build cleanly
npm run build

# 2. Inspect the tarball without publishing
npm pack --dry-run
```

Expected `npm pack --dry-run` output for a module:

```
npm notice === Tarball Contents ===
npm notice 1.2kB  README.md
npm notice 1.0kB  LICENSE
npm notice 3.4kB  dist/index.js
npm notice 2.1kB  dist/index.d.ts
npm notice 1.8kB  dist/index.js.map
npm notice  512B  dist/docs/my-module.txt   ← MUST appear
npm notice 4.2kB  dist/services/MyService.js
npm notice ...
npm notice === Tarball Details ===
npm notice name:          @scope/my-module
npm notice version:       1.0.0
npm notice filename:      scope-my-module-1.0.0.tgz
npm notice total files:   18
```

**The tarball must NOT contain:**
- `src/` directory — means `"files"` is wrong or not set
- `node_modules/` — means `.npmignore` or `"files"` is wrong
- `cfg/config.json` with real credentials — check before publishing
- `*.env` files

**The tarball MUST contain:**
- `dist/index.js` and `dist/index.d.ts`
- `dist/docs/<alias>.txt` — without this, `help()` prints nothing
- `README.md` and `LICENSE`

### First publish of a scoped package

```bash
# Scoped packages (@kozen/*, @scope/*) default to private — must pass --access public
npm publish --access public

# Or use the publishConfig in package.json (already set) and run without the flag:
npm publish
```

The `publishConfig.access: "public"` in `package.json` removes the need for `--access public`
on every publish. Verify it is set before the first publish.

### Subsequent publishes — version bump strategy

Kozen follows semantic versioning (`MAJOR.MINOR.PATCH`):

| Change type | Version bump | Command |
|---|---|---|
| Bug fix, no API change | Patch: `1.0.0 → 1.0.1` | `npm version patch` |
| New feature, backwards-compatible | Minor: `1.0.0 → 1.1.0` | `npm version minor` |
| Breaking API change | Major: `1.0.0 → 2.0.0` | `npm version major` |
| Pre-release / RC | Pre-release: `1.0.0 → 1.1.0-rc.0` | `npm version preminor` |

```bash
# Standard patch release
npm version patch          # bumps package.json version and creates a git tag
npm run build              # rebuild with the new version embedded in package.json
npm publish                # publish the new version

# Or in one step using the publish script:
npm version patch && npm run publish
```

`npm version` automatically:
1. Updates `package.json` version
2. Commits the change with message `v1.0.1`
3. Creates a git tag `v1.0.1`

Push the tag after publishing:
```bash
git push && git push --tags
```

### Post-publish verification

```bash
# Verify the published version is visible on npm
npm view @scope/my-module versions
# → [ '1.0.0', '1.0.1' ]

# Install in a clean temporary directory to verify the package works end-to-end
mkdir /tmp/kozen-test && cd /tmp/kozen-test
npm init -y
npm install @kozen/engine @scope/my-module
npx kozen --moduleLoad=@scope/my-module --action=my-module:help
```

If `help()` prints the module description, all three things are verified:
1. The package installed correctly
2. The `dist/docs/<alias>.txt` file was included in the tarball
3. The module entry point and IoC registration work

### `publish` script in package.json

The `"publish": "npm run build && npm publish"` script in `package.json` runs the build
before every publish, preventing accidental publication of stale compiled output:

```bash
npm run publish    # builds then publishes — use this for all releases
```

**Do not use `npm run publish` in CI/CD** if your pipeline already runs the build step
separately — it would build twice. Use `npm publish` directly in that case.

---

## 18. Complete minimal module example

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

## 19. Checklist before publishing

### Module code

### Module code

| Check | Why |
|---|---|
| `this.metadata.alias` is set and unique across loaded modules | Action routing breaks without it |
| `package.json` `homepage` field is set | Printed by `help()` as the docs URL |
| All `ioc.json` `path` values are relative strings (not absolute) | `fix()` converts them to absolute at runtime |
| `this.fix(dep)` called before returning from `register()` | Without it, npm-installed modules fail with `Cannot find module` |
| Token format follows `alias:service`, `alias:controller:cli`, `alias:controller:mcp` | Engine action routing requires this exact format |
| `cli.json` only loaded in `'cli'` branch; `mcp.json` only in `'mcp'` | Prevents loading unused controllers |
| `@kozen/engine` is in `dependencies` (not `devDependencies`) | Required at runtime in any project that installs the module |
| No `bin` field in `package.json` | Modules use the engine binary — a module bin breaks tsconfig rootDir |

### package.json fields

| Check | Why |
|---|---|
| `"files": ["dist/*", "LICENSE", "README.md"]` | Prevents `src/`, `cfg/`, and credentials from being published |
| `publishConfig.access: "public"` set | Scoped packages default to private on npm |
| `"engines": { "node": ">=18" }` present | Documents the runtime requirement; CI tools enforce it |
| `@types/node` version in `devDependencies` matches `engines.node` major | Mismatched types cause spurious build errors |
| `zod` in `dependencies` only if the module defines MCP tools | Adding it unconditionally bloats packages that don't use Zod |
| `test` script present (placeholder or real) | `npm test` must exit cleanly — CI pipelines call it unconditionally |
| `bugs.url` and `homepage` fields set | `help()` and `npm info` surface these to users |
| `repository.url` uses `git+https://` prefix | Standard npm convention; some tools parse it |

### Documentation

| Check | Why |
|---|---|
| `src/docs/<alias>.txt` exists | `help()` action reads it; missing file → silent empty output |
| `.txt` file name matches `this.metadata.alias` exactly | `FileService` looks up by alias name |
| `src/docs/<alias>.txt` is included in the `copy:txt` glob | Without the copy step, `dist/docs/` will not have the file |
| `README.md` is present at project root | Included in the npm tarball; displayed on npmjs.com |

### Build

| Check | Why |
|---|---|
| `tsconfig.json` `include` is `["src/**/*.ts"]` only | Adding `bin/**/*.ts` shifts rootDir and breaks the `main` field |
| `tsconfig.json` has `"resolveJsonModule": true` | Required for `import ioc from './configs/ioc.json'` |
| `dist/index.js` exists after `npm run build` | The `main` field must resolve |
| `dist/index.d.ts` exists after `npm run build` | TypeScript consumers need the declarations |
| `dist/docs/<alias>.txt` exists after `npm run build` | Confirms `copy:txt` script ran correctly |
| `dist/` does NOT contain a `src/` subdirectory | Would mean tsconfig `include` has `bin/**/*.ts` — fix it |

### npm publish

| Check | Why |
|---|---|
| `npm pack --dry-run` output does not include `src/` or `node_modules/` | Final sanity check before committing to publish |
| `npm pack --dry-run` output includes `dist/docs/<alias>.txt` | Confirms the help file will be in the published package |
| Version in `package.json` is bumped from the last published version | `npm publish` fails if the version already exists on npm |
| Git working tree is clean and version commit/tag was pushed | Keeps git history aligned with npm versions |
| Post-publish: `npx kozen --moduleLoad=@scope/my-module --action=my-module:help` works in a clean directory | End-to-end confirmation the published package is functional |
