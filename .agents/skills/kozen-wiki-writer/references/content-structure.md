---
name: Kozen Wiki — Content Structure
description: >
  What to write on each wiki page type (Home, Get-Started, Configuration, CLI, MCP, Delegate,
  API, Kozen-Integration, POLICY), the source analysis checklist for a Kozen module (what to
  read in the source code before writing), the no-duplication rule with linking patterns,
  escalated progressive content strategy, bibliography and references requirements, and
  licence/disclaimer page structure.
category: kozen-wiki-writer
tags:
  - wiki
  - content
  - page-types
  - source-analysis
  - no-duplication
  - progressive-disclosure
  - licence
  - disclaimer
  - bibliography
---

# Kozen Wiki — Content Structure

---

## Source analysis checklist

Read these files in the module source code **before** writing any wiki page. Never invent
API signatures, options, or behaviour — derive everything from the source.

| Source file | What to extract |
|---|---|
| `src/index.ts` | Every exported class, interface, type, and function. This becomes the API page and the exports table on Home.md. Do not skip any export. |
| `src/docs/*.txt` | Authoritative CLI help text. Reuse the action names, flag names, env var names, and examples verbatim. Do not rewrite what is already correct here. |
| `src/configs/ioc.json` | Registered service tokens, lifetimes, and dependency keys. Explains what the IoC container provides without running the code. |
| `src/configs/cli.json` | CLI controller token and its injected dependencies. Confirms which services the CLI layer depends on. |
| `src/configs/mcp.json` | MCP controller token. Confirms whether the module has MCP support at all. If this file does not exist, the module has no MCP page. |
| `package.json` | Module name, version, description, homepage URL, author, licence. All of these appear in Home.md and the POLICY page. |
| `src/models/*.ts` | Interface and type definitions for public API documentation. |
| `src/services/*.ts` | Service class signatures and method documentation. |
| `src/controllers/*.ts` | CLI/MCP controller method names = CLI actions. Each public method is a `--action` entry. |
| `README.md` (if present) | Quick start and installation examples. Reuse, do not duplicate. Link to it from Get-Started.md if relevant. |

### Critical: the entry point (`src/index.ts`)

The entry point is the single most important file for wiki documentation. It declares
the public contract of the module. Every export must appear on the API/Integration page.

Checklist for `src/index.ts`:
- [ ] Module class (the `KzModule` subclass) — document `metadata.alias`, `register()` behaviour
- [ ] All service classes — document in API page
- [ ] All interface exports (`I*`) — document fields in API page
- [ ] All type exports — document in API page
- [ ] Controller exports (rarely needed by users, but note they exist)
- [ ] Re-exported utilities from `@kozen/engine` (if any)

---

## No-duplication rule

**Never write the same information twice.** If a concept is explained on one page, all
other pages link to it.

Examples of correct cross-linking instead of duplication:

```markdown
# Installation
See [Get Started](https://github.com/<org>/<repo>/wiki/Get-Started) for installation
instructions.
```

```markdown
# Environment Variables
For the full list of engine-level variables (KOZEN_STACK, KOZEN_LOG_LEVEL, etc.),
see the [Kozen Engine Configuration](https://github.com/kozen-labs/engine/wiki/Configuration).
This page covers only the variables specific to `@kozen/trigger`.
```

```markdown
# URI Resolution
The URI resolution priority is: `uri` → `uriEnv` → built from parts.
See [Configuration](https://github.com/<org>/<repo>/wiki/Configuration) for full details.
```

If you find yourself repeating a table, a code block, or a paragraph across two pages,
stop and turn one of them into a link.

---

## Escalated / progressive content strategy

Each page in the wiki assumes the reader has completed all previous pages. Content
escalates in complexity as the reader moves through the wiki:

```
Home          ← Overview, value proposition, no technical depth
  ↓
Get-Started   ← Install + first working command (zero-to-success in one page)
  ↓
Configuration ← All options, env vars, config files, advanced parameters
  ↓
CLI / MCP     ← All actions and tools with full flag tables and examples
  ↓
Delegate/API  ← Full programmatic API, interface reference, edge cases
  ↓
Integration   ← How to compose with the Kozen IoC container and other modules
  ↓
POLICY        ← Legal, licence, disclaimer — final page
```

**Rule:** Never put advanced content on an early page. If a reader needs to know something
to understand a section, either it was on a previous page (link to it) or it is an
incorrect placement.

---

## Page type templates

### Home.md = README.md parity rule

`Home.md` (GitHub Wiki) and `README.md` (repository root) must contain **identical content**.

Why: npm, PyPI, and crates.io display `README.md` directly. GitHub displays `Home.md` as the
wiki landing page. A developer discovering the module through any of those surfaces must see
the same high-level overview and be able to navigate to the full documentation from there.

**Workflow:**
1. Write `Home.md` first, following the template below.
2. Copy the finished content verbatim into `README.md`.
3. Replace any GitHub wiki links in `README.md` with the same absolute wiki URLs — they work
   from npm and GitHub alike.
4. When either file changes, sync the other immediately.

**What must stay identical:**
- Module name and one-sentence definition
- Key Features list
- Why Use This section
- Installation snippet
- References / navigation links
- Disclaimer blockquote

**What may differ only in link format:** GitHub Wiki renders wiki-relative links; README.md
must use full absolute URLs. In practice, since both should already use absolute GitHub wiki
URLs, no adjustment is needed.

---

### Home.md

`Home.md` is the entry point for developers on GitHub, npm, and PyPI. Keep it minimal:
a high-level overview that earns the reader's trust and guides them to the right wiki page.
No API details, no configuration tables, no flag references — those belong on dedicated pages.

**Minimal content rule:** If a sentence on `Home.md` requires the reader to know what a flag
or interface field does, move that sentence to the appropriate dedicated page and replace it
with a link.

Required sections (in order):

1. **Module name and one-sentence definition** — what it does and what problem it solves.
2. **🌟 Key Features** — 4 to 8 bullet points with emojis. Concrete, not generic.
3. **⚡ Why Use This?** — 2 to 4 sentences. The use case in plain language.
4. **📦 Installation** — npm install command, minimal .env, first command. Maximum 15 lines.
5. **📚 References** — links to all other wiki pages, the GitHub repo, npm page, and external docs.
6. **Disclaimer blockquote** — one line linking to POLICY.md.
7. **Navigation footer**

Do NOT put configuration details, API signatures, or CLI flag tables on Home.md.

### Get-Started.md

Required sections:

1. **Prerequisites** — Node.js version, required environment variables.
2. **Installation** — `npm install @scope/module-name`.
3. **Minimal configuration** — the smallest `.env` or config that makes the module work.
4. **First command** — one copy-paste CLI invocation that produces visible output.
5. **Verify it worked** — what to look for in the output to confirm success.
6. **Next steps** — links to Configuration.md for full options, and CLI/MCP pages.
7. **Navigation footer**

Get-Started must be completable in under 5 minutes for a developer who has never used Kozen.

### Configuration.md

Required sections:

1. **Configuration layers** — priority order (CLI flag > env var > .env file > config.json > default). Link to Kozen Engine wiki for engine-level vars; this page only covers module-specific vars.
2. **Environment variables table** — all `MODULE_*` env vars with: name, CLI equivalent, default, description.
3. **`.env` file example** — a realistic, copyable example covering the common case.
4. **`cfg/config.json` snippet** — if the module has standalone config.
5. **Advanced options** — driver comparisons, special flags, backend-specific settings.
6. **Security notes** — what must never be committed, how to rotate keys.
7. **Navigation footer**

### CLI page (`[Module]-via-CLI.md` or `CLI.md`)

Required sections:

1. **Available actions table** — action name, description, required/optional.
2. **Per-action detail** — for each action: full invocation syntax, all flags, env var fallbacks, example output.
3. **Full flags reference table** — every flag, env var, default, description.
4. **Common patterns** — 3 to 5 realistic examples covering different use cases.
5. **Navigation footer**

Source the action names from `src/controllers/*CLIController.ts` public method names.
Source the flag names from the controller's `fill()` method and `src/docs/*.txt`.

### MCP page (`[Module]-via-MCP.md` or `MCP.md`)

Required sections:

1. **What MCP tools are registered** — tool names and one-line descriptions.
2. **Per-tool detail** — name, description, full input schema with types and required/optional.
3. **MCP server configuration** — JSON block for VS Code / Claude Desktop `mcpServers`.
4. **Usage examples** — 2 to 3 example prompts the user might give an LLM that would trigger these tools.
5. **Navigation footer**

Source tool names from `src/controllers/*MCPController.ts`, the `server.registerTool()` calls.
Only write this page if `src/configs/mcp.json` exists.

### Delegate page (`Delegate.md`)

For `@kozen/trigger` specifically and any future module that uses the delegate pattern.

Required sections:

1. **What a delegate is** — one-paragraph definition.
2. **Handler dispatch rules** — the priority order (operation-specific → catch-all).
3. **Handler name reference table** — all valid export names mapped to MongoDB `operationType`.
4. **ITriggerTools reference** — every field with type and description.
5. **ESM and CJS format examples** — complete, runnable delegate files.
6. **Direct MongoDB access pattern** — using `tools.db` and `tools.collection`.
7. **Service resolution pattern** — using `tools.assistant?.resolve()`.
8. **Module system detection rules** — how Kozen chooses ESM vs CJS.
9. **Navigation footer**

### API page (`API.md` or `Kozen-Module-Development.md`)

Required sections:

1. **All public exports** — code block listing every `import { ... } from '@scope/module'`.
2. **Module class** — constructor signature, `register()` behaviour, IoC tokens registered.
3. **Per-interface** — all fields with types, descriptions, and whether required/optional.
4. **Per-service class** — constructor signature, all public methods with full signatures.
5. **Programmatic usage** — TypeScript examples: with Kozen IoC, standalone without Kozen.
6. **Navigation footer**

The "all public exports" code block is derived directly from `src/index.ts`. Do not guess.

### Kozen-Integration page (`Kozen-Integration.md`)

This page explains how to load the module into a running Kozen application via `--moduleLoad`.

Required sections:

1. **Module loading** — `--moduleLoad=@scope/module-name` CLI flag.
2. **IoC token reference** — table of all tokens registered by this module (from configs/*.json).
3. **Composing with other modules** — example loading two modules together.
4. **`.env` integration pattern** — KOZEN_MODULE_LOAD env var.
5. **Navigation footer**

### POLICY.md (Disclaimer and Licence)

This page is mandatory. It protects the authors legally and informs users of usage terms.

Required sections (in order):

1. **⚖️ [Module Name] Disclaimer and Usage Policy** — H1 with scale emoji.
2. **Open Source Distribution** — state the licence (MIT, ISC, Apache-2.0 — from package.json). Include the full licence name and a link to the licence text if possible.
3. **Purpose and Vision** — 2 to 3 sentences on what the project aims to do.
4. **Officially Maintained Modules** — list of modules maintained by the same team.
5. **Contributions** — brief note on contribution process.
6. **Liability Disclaimer** — standard disclaimer: provided as-is, no warranties, authors not liable for damages. Use plain language, not legalese.
7. **Final Notes** — contact / issues tracker link.
8. **Navigation footer**

Template for the Liability Disclaimer section:

```markdown
## Liability Disclaimer

This software is provided "as is", without warranty of any kind, express or implied,
including but not limited to warranties of merchantability, fitness for a particular
purpose, and non-infringement. In no event shall the authors or copyright holders be
liable for any claim, damages, or other liability arising from the use of this software.

Use of this module in production environments is at the user's own risk. The authors
recommend thorough testing in non-production environments before deployment.
```

---

## Bibliography and References conventions

### When a References section is required

- **Home.md** — always, with links to all wiki pages + external resources.
- **Get-Started.md** — always, with links to Configuration and the Kozen Engine wiki.
- **Delegate.md / API.md** — when citing MongoDB driver documentation or external specs.
- **POLICY.md** — when citing the licence text or contributing guidelines.
- **All other pages** — include a References section if the page cites any external resource.

### Format

Follow the `ks-technical-article-writer` pattern for wiki pages (markdown links format):

```markdown
## 📚 References

- [Kozen Engine Wiki](https://github.com/kozen-labs/engine/wiki) — core framework documentation
- [MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/) — official MongoDB documentation
- [npm: @kozen/trigger](https://www.npmjs.com/package/@kozen/trigger) — package registry entry
- [Disclaimer and Usage Policy](https://github.com/<org>/<repo>/wiki/POLICY) — usage terms and licence
- [Contributing Guidelines](https://github.com/<org>/<repo>/blob/main/CONTRIBUTING.md)
```

Sources to always include on Home.md:
- The Kozen Engine wiki
- This module's npm page
- This module's GitHub repository
- The POLICY page
- The MongoDB documentation page most relevant to the module's domain

---

## Licence and disclaimer placement rules

| Page | What to include |
|---|---|
| `POLICY.md` | Full licence notice, liability disclaimer, contribution guidelines |
| `Home.md` | Blockquote linking to POLICY.md — one line only |
| `Get-Started.md` | Optional: one-line reference to POLICY.md in the References section |
| All other pages | No licence text; link to POLICY.md from References only |

Never repeat the full licence text or liability disclaimer on any page other than POLICY.md.

---

## Banner image convention

Home.md may include a banner image as the very first element (before the H1) if the
module repository has one:

```markdown
![Kozen Trigger Banner](https://raw.githubusercontent.com/<org>/<repo>/main/banner.png)

# 🚀 Kozen Trigger
```

If no banner exists, omit the image entirely. Never use a placeholder image.

---

## Checklist before marking a wiki complete

- [ ] Every page has a navigation footer with correct Previous/Next links
- [ ] Home.md has the disclaimer blockquote linking to POLICY.md
- [ ] POLICY.md exists and includes licence (from package.json `license` field)
- [ ] All internal links use absolute GitHub wiki URLs (Mode A) or relative paths (Mode B)
- [ ] No content is duplicated across pages — check for copy-pasted paragraphs
- [ ] All API signatures were verified against `src/index.ts`
- [ ] All CLI action names were verified against the controller's public methods
- [ ] All env var names were verified against the `src/docs/*.txt` help file
- [ ] `src/configs/mcp.json` existence was checked before writing an MCP page
- [ ] Emojis appear only on H1 and H2 headings
- [ ] Every page that cites an external resource has a References section
- [ ] POLICY.md licence matches the `license` field in `package.json`
