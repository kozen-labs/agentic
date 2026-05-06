---
name: kozen-engine
description: >
  Expert-level guide for the Kozen Task Execution Framework (@kozen/engine). Covers core
  architecture (Applications, Modules, Controllers, Components, IoC container), all CLI and
  MCP application interfaces, full module development lifecycle (creating KzModule subclasses,
  registering IoC dependencies, building CLI and MCP controllers, implementing services),
  configuration schemas (config.json, ioc.json, cli.json, mcp.json), the structured logging
  system, npm publishing patterns for distributable Kozen modules, and contribution guidelines.
  This skill is the primary reference for anyone building new tools, modules, or extensions
  for the Kozen ecosystem.
created: 2026-05-03
updated: 2026-05-06
---

# Kozen Engine — Framework Reference

Kozen (`@kozen/engine`) is a lightweight **Task Execution Framework** built on Node.js that
provides CLI, MCP (Model Context Protocol), REST, and SDK interfaces backed by an IoC
(Inversion of Control) container and a plugin module system. Any domain capability can be
packaged as a Kozen module, published to npm, and loaded at runtime without modifying the
core engine.

---

## Routing table

| Signal | Reference |
|---|---|
| design principles, KISS, SOLID, SoC, OOP, GoF, GRASP, Strategy pattern, yo-yo problem, class hierarchy, utility function vs service, IoC dynamic loading, code quality, simplicity, maintainability, efficiency, duplication, composition over inheritance, circular import, circular dependency, import cycle, cyclic dependency, import direction, layer architecture, one-way imports, async, await, event loop, non-blocking, fs/promises, readFileSync, synchronous I/O, blocking I/O, Node.js performance | `references/module-development.md` |
| what is Kozen, Kozen overview, Kozen architecture, framework concepts, what is a Module, what is a Controller, what is a Component, what is an Application, what is a Service, when to use Module vs Controller vs Service vs Component, IoC container, inversion of control, dependency injection, how IoC works, Awilix, registration strategies, class ref value auto function, lifetime singleton transient scoped, fix() function, lazy vs eager resolution, IoC flexibility, composing modules via IoC, replacing implementations, KzModule, KzController, KzApplication, KzComponent, BaseService, flow ID, IConfig, IArgs, IModule, IMetadata, module loading lifecycle, token naming conventions | `references/concepts.md` |
| create module, develop module, build module, new Kozen module from scratch, KzModule subclass, extend KzModule, register() function, what does register() do, why register() exists, switch config.type, ioc.json, cli.json, mcp.json, why three config files, why split ioc.json and cli.json, why package.json is loaded at runtime, metadata from package.json, module constructor, alias, fix(), dependency map, IDependency, IoC registration, fill() method, argument parsing, env var fallbacks, help() action, docs txt file, plain text documentation, src/docs, module documentation structure, BaseService, service class, create service, create controller, CLI controller, MCP controller, MCPController, server.registerTool, Zod schema, tool naming, publish module, npm publish kozen, npm login, npm pack, npm version, publishConfig, scoped package, @scope npm, dist structure, tsconfig rootDir, main field, types field, copy:txt, entry point path, bin field kozen, why no bin module, kozen binary modules, scripts publish versions, version bump patch minor major, post-publish verify, module checklist, how to build a Kozen tool, complete module example, JSDoc, code comments, comment standard, jsdoc style, how to comment, @param @returns @deprecated @throws, no examples in comments, brief comments, logging in module, how to log in service, how to log in controller, logger injection, logger:service token, ILogEntry, ILogLevel, flow ID, this.getId, this.wait, logger.stack, src naming convention, VCategory, circular import, circular dependency, import cycle, cyclic dependency, import direction, layer architecture, one-way imports | `references/module-development.md` |
| config.json, kozen.json, configuration schema, environment variables, KOZEN_MODULE_LOAD, KOZEN_STACK, KOZEN_PROJECT, KOZEN_CONFIG, KOZEN_APP_TYPE, KOZEN_TEMPLATE, template JSON, pipeline template, template component, stack configuration, IDependency config, singleton transient scoped, .env file, multi-environment config, cfg/config.json, env var naming convention, KOZEN prefix, module env var prefix, KOZEN_TRIGGER_, KOZEN_IAM_, module-specific env var, when to reuse env var, AWS env var, MDB env var | `references/configuration.md` |
| CLI app, kozen CLI, --action, --moduleLoad, --type=mcp, MCP app, MCP server, MCP integration, npx kozen, kozen binary, module loading CLI, module loading env var, KOZEN_MODULE_LOAD env, load modules, CLI options, MCP JSON config, VS Code MCP, logger CLI, logger service, LoggerService API, structured logging, KOZEN_LOG_LEVEL, KOZEN_LOG_TYPE, KOZEN_LOG_CONSOLE_ENABLED, KOZEN_LOG_MDB_ENABLED, KOZEN_LOG_LEVEL_REMOTE, console processor, MongoDB processor, file processor, hybrid processor, log destinations, configure logger, logger.configure, logger.add, logger.stack flush, await Promise.all logger.stack | `references/cli-mcp.md` |
| public exports @kozen/engine, what does @kozen/engine export, use kozen as a library, use Kozen as a platform, IoC class methods, IIoC interface, register dependencies, resolve service, get service, resolveSync, unregister, IoC standalone, BaseService constructor, KzModule methods fix register requires, KzController methods fill configure load, CLIController, MCPController register server, KzApplication init start, getID utility, readFrom utility, Env class, MdbClient IMdbClientOpt, Logger ILogLevel, JSONT EnumUtl VCategory, IComponent IComponentInput IComponentOutput IModuleOpt, embed Kozen module, CLI platform, MCP platform, library pattern, IoC standalone pattern | `references/public-api.md` |

---

## When to use this skill

- Understanding the Kozen framework's core architecture before contributing
- Building a new Kozen module from scratch (the full development workflow)
- Integrating an existing service domain into the Kozen ecosystem
- Connecting Kozen to an AI assistant via MCP
- Setting up environment variables, config files, or IoC registration
- Publishing a Kozen module as an npm package
- Debugging module loading, IoC resolution, or controller dispatch issues

---

## Ecosystem overview

`@kozen/engine` has no dependency on any specific backend. It is a
general-purpose Task Execution Framework. The modules listed below are domain integrations
built on top of the engine — they are optional and independently published.

```
@kozen/engine             ← core framework; installs the kozen CLI binary;
@kozen/secret             ← optional module: AWS Secrets Manager + MongoDB CSFLE
@kozen/trigger            ← optional module: self-hosted MongoDB Change Stream triggers
@kozen/iam-rectification  ← optional module: MongoDB IAM permission validation
```

A module can integrate any service or API. MongoDB appears in several existing modules
because the original use cases required it — not because the framework requires it.

All module packages follow the same structural contract (see `references/module-development.md`).

---

## Working strategy — analyze, align, plan, execute

Every task involving a Kozen module follows this four-step sequence. **Never skip to
implementation without completing steps 1 and 2.**

### Step 1 — Analyze before touching anything

Read all relevant artifacts before writing a single line of code or configuration:

| Artefact | What to verify |
|---|---|
| `src/index.ts` | Exported classes, interfaces, and types — public API surface |
| `src/configs/ioc.json`, `*.json` | Registered tokens, lifetimes, dependency keys |
| `src/controllers/*.ts` | Actions, `fill()` overrides, env-var fallbacks |
| `src/services/*.ts` | Business logic, constructor signatures, error handling |
| `src/models/*.ts` | Existing interfaces — avoid duplication |
| `src/docs/<alias>.txt` | Documented flags, env vars, default values |
| `package.json` | Name, version, scripts, `files`, `main`, `types` |
| `tsconfig.json` | `include`, `outDir`, path aliases |
| Existing reference modules (`@kozen/secret`, `@kozen/trigger`, `@kozen/iam-rectification`) | Established patterns to reuse or align with |

### Step 2 — Surface ambiguities before proceeding

After the analysis, identify every point that is unclear, missing, or inconsistent. For each
one, ask a focused question. Do not assume an answer and proceed — a wrong assumption
discovered after implementation costs far more than a clarifying question asked upfront.

Common triggers for a clarifying question:

- A required field or parameter is not documented anywhere in the source.
- Two existing modules follow different conventions for the same thing (e.g., different token
  naming patterns, different env-var prefixes) — ask which one is authoritative.
- The requested change conflicts with an existing pattern or a rule in this skill.
- It is unclear whether a new class should be a service, a utility function, or a separate npm
  package.
- The lifetime (`singleton` / `transient`) of a new IoC registration is ambiguous.

Ask all questions together in one message so the user can answer them in a single round.

### Step 3 — Write a plan and get approval

Once the analysis is complete and all ambiguities are resolved, write a plan file:

```
tmp/plan-<short-description>.md
```

The plan must include:

1. **Summary** — one paragraph describing what will change and why.
2. **Files to create or modify** — listed with the reason for each change.
3. **Key decisions** — any non-obvious choice made (lifetime, naming, file placement) with
   a brief justification.
4. **Out of scope** — anything explicitly excluded to prevent scope creep.

Present the plan to the user and wait for explicit approval before writing any production
code. If the user requests changes to the plan, update the file and present it again.

### Step 4 — Apply changes, then clean up

Once the plan is approved:

1. Apply all changes exactly as described in the plan.
2. Verify the checklist in Section 19 of `references/module-development.md`.
3. Delete the plan file from `tmp/` — it is a working artefact, not a permanent document.

---

## Do & Don't

**Do:**
- Use JSDoc format for all public classes and methods; keep every comment as simple as possible; include arguments and return types in the JSDoc, even if they can be inferred from TypeScript — it improves readability and editor tooltips.
- Document the WHY (non-obvious constraints, side effects) — never restate what the name already says.
- Apply KISS first: the simplest correct solution wins over any pattern.
- Cap class hierarchies at two levels (base + one subclass) to avoid the yo-yo problem.
- Use the Strategy pattern (one class per variant) whenever behavior can vary by configuration.
- Load all dynamic classes and resources through the IoC container — never `new MyClass()` inside a service or controller body.
- Promote a utility function to an IoC service when it needs injected state (connection, logger, config).
- Always extend `KzModule` as the module entry point.
- Register IoC dependencies in separate JSON config files (`ioc.json`, `cli.json`, `mcp.json`).
- Call `this.fix(dep)` on the dependency map before returning from `register()`.
- Use `BaseService` as the base class for all service implementations.
- Carry the `flow` ID through every log call for tracing.
- Load metadata from `package.json` inside the module constructor.
- Set `publishConfig.access: "public"` in `package.json` for scoped npm packages.
- Use `async`/`await` for every I/O operation — file system (`fs/promises`), MongoDB, network, crypto, child processes.

**Don't:**
- Never name a module-specific env var without the `KOZEN_<MODULE_ALIAS>_` prefix — unless reusing an established ecosystem variable (`AWS_*`, `MDB_*`).
- Never put usage examples inside JSDoc comments — they belong in `src/docs/<alias>.txt`.
- Never collapse JSDoc to a single line (`/** text */`) — always use the multi-line block format.
- Never create circular imports — imports must flow one way: `models` ← `services` ← `controllers` ← `index.ts`. Cross-service dependencies must be resolved through IoC tokens, not direct class imports. Treat any import cycle as a build error.
- Never build a class hierarchy deeper than two levels — use composition or Strategy instead.
- Never instantiate service or controller classes with `new` inside module code — use IoC.
- Never duplicate domain logic across services — extract shared logic to a service registered in `ioc.json` or a dedicated npm package.
- Never instantiate services directly; always inject via the IoC container.
- Never merge `cli.json` or `mcp.json` for the wrong runtime type inside `register()`.
- Never hardcode connection strings or credentials inside module source code.
- Never skip `this.fix(dep)` — it resolves relative `path` entries to absolute paths.
- Never add a `bin` field to a module's `package.json` — modules use the engine binary (`npx kozen`); adding a `bin` also shifts the tsconfig `rootDir` and breaks `dist/index.js`.
- Never add `bin/**/*.ts` to tsconfig `include` in a module — only `src/**/*.ts`.
- Never publish without running `npm pack --dry-run` first — verify `dist/docs/<alias>.txt` is included and `src/` is not.
- Never use synchronous I/O APIs (`readFileSync`, `writeFileSync`, `execSync`, etc.) — always use their async counterparts from `fs/promises` and similar. The only narrow exception is module startup (inside `register()` or a constructor) for a single small config file read, and only when justified with a JSDoc comment.
