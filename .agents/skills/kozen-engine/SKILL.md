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
updated: 2026-05-03
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
| what is Kozen, Kozen overview, Kozen architecture, framework concepts, what is a Module, what is a Controller, what is a Component, what is an Application, what is a Service, when to use Module vs Controller vs Service vs Component, IoC container, inversion of control, dependency injection, how IoC works, Awilix, registration strategies, class ref value auto function, lifetime singleton transient scoped, fix() function, lazy vs eager resolution, IoC flexibility, composing modules via IoC, replacing implementations, KzModule, KzController, KzApplication, KzComponent, BaseService, flow ID, IConfig, IArgs, IModule, IMetadata, module loading lifecycle, token naming conventions | `references/concepts.md` |
| create module, develop module, build module, new Kozen module from scratch, KzModule subclass, extend KzModule, register() function, what does register() do, why register() exists, switch config.type, ioc.json, cli.json, mcp.json, why three config files, why split ioc.json and cli.json, why package.json is loaded at runtime, metadata from package.json, module constructor, alias, fix(), dependency map, IDependency, IoC registration, fill() method, argument parsing, env var fallbacks, help() action, docs txt file, plain text documentation, src/docs, module documentation structure, BaseService, service class, create service, create controller, CLI controller, MCP controller, MCPController, server.registerTool, Zod schema, tool naming, publish module, npm publish kozen, npm login, npm pack, npm version, publishConfig, scoped package, @scope npm, dist structure, tsconfig rootDir, main field, types field, copy:txt, entry point path, bin field kozen, why no bin module, kozen binary modules, scripts publish versions, version bump patch minor major, post-publish verify, module checklist, how to build a Kozen tool, complete module example | `references/module-development.md` |
| config.json, kozen.json, configuration schema, environment variables, KOZEN_MODULE_LOAD, KOZEN_STACK, KOZEN_PROJECT, KOZEN_CONFIG, KOZEN_APP_TYPE, KOZEN_TEMPLATE, template JSON, pipeline template, template component, stack configuration, IDependency config, singleton transient scoped, .env file, multi-environment config, cfg/config.json | `references/configuration.md` |
| CLI app, kozen CLI, --action, --moduleLoad, --type=mcp, MCP app, MCP server, MCP integration, npx kozen, kozen binary, module loading CLI, module loading env var, KOZEN_MODULE_LOAD env, load modules, CLI options, MCP JSON config, VS Code MCP, logger CLI, logger service, ILogLevel, LoggerService, structured logging, flow logging, KOZEN_LOG_LEVEL, KOZEN_LOG_TYPE, console processor, MongoDB processor, log destinations | `references/cli-mcp.md` |
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

```
@kozen/engine          ← npm package; installs the kozen CLI binary
@kozen/secret          ← module: AWS + MongoDB CSFLE secret management
@kozen/trigger         ← module: self-hosted MongoDB Change Stream triggers
@kozen/iam-rectification  ← module: MongoDB IAM role/permission validation
```

All module packages follow the same structural contract (see `references/module-development.md`).

---

## Do & Don't

**Do:**
- Always extend `KzModule` as the module entry point.
- Register IoC dependencies in separate JSON config files (`ioc.json`, `cli.json`, `mcp.json`).
- Call `this.fix(dep)` on the dependency map before returning from `register()`.
- Use `BaseService` as the base class for all service implementations.
- Carry the `flow` ID through every log call for tracing.
- Load metadata from `package.json` inside the module constructor.
- Set `publishConfig.access: "public"` in `package.json` for scoped npm packages.

**Don't:**
- Never instantiate services directly; always inject via the IoC container.
- Never merge `cli.json` or `mcp.json` for the wrong runtime type inside `register()`.
- Never hardcode connection strings or credentials inside module source code.
- Never skip `this.fix(dep)` — it resolves relative `path` entries to absolute paths.
- Never add a `bin` field to a module's `package.json` — modules use the engine binary (`npx kozen`); adding a `bin` also shifts the tsconfig `rootDir` and breaks `dist/index.js`.
- Never add `bin/**/*.ts` to tsconfig `include` in a module — only `src/**/*.ts`.
- Never publish without running `npm pack --dry-run` first — verify `dist/docs/<alias>.txt` is included and `src/` is not.
