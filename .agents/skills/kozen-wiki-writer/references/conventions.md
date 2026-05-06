---
name: Kozen Wiki — Format Conventions
description: >
  All formatting rules for Kozen wiki pages: exact navigation footer syntax, emoji-heading
  conventions with the standard emoji map, file naming (PascalCase + hyphens), absolute
  GitHub wiki link format, code block language tags, table structure, tone and voice, and
  page ordering within a module wiki. Based on patterns from engine.wiki, trigger.wiki,
  secret.wiki, and iam-rectification.wiki.
category: kozen-wiki-writer
tags:
  - wiki
  - conventions
  - navigation-footer
  - emojis
  - formatting
  - tone
---

# Kozen Wiki — Format Conventions

---

## Navigation footer (mandatory on every page)

Every page ends with a navigation footer. This is the single most important convention —
it is what makes the wiki feel like a coherent document rather than isolated files.

### Exact format

```
← Previous: [Page Title](https://github.com/<org>/<repo>/wiki/<PageName>) | Next: [Page Title](https://github.com/<org>/<repo>/wiki/<PageName>) →
```

Rules:
- One blank line above the footer, then a `---` horizontal rule, then the footer on the next line.
- Use `—` (em dash) when there is no previous or next page — this is the **only** permitted use of `—` in wiki content.
- The Home page has no previous: `← Previous: — | Next: [Page Title](URL) →`
- The last page has no next: `← Previous: [Page Title](URL) | Next: — →`
- The Home page URL is the wiki root, not `/Home`: `https://github.com/<org>/<repo>/wiki`

### Examples from existing wikis

```markdown
← Previous: — | Next: [Introduction](https://github.com/kozen-labs/engine/wiki/Introduction) →
```
```markdown
← Previous: [Configuration](https://github.com/kozen-labs/engine/wiki/Configuration) | Next: [Application CLI](https://github.com/kozen-labs/engine/wiki/App-CLI) →
```
```markdown
← Previous: [Get Started](https://github.com/kozen-labs/trigger/wiki/Get-Started) | Next: [Config](https://github.com/kozen-labs/trigger/wiki/Config) →
```
```markdown
← Previous: [Kozen Integration](https://github.com/kozen-labs/iam-rectification/wiki/Kozen-Integration) | Next: — →
```

### POLICY page footer convention

The POLICY page always links back to Home as its Previous, since it is typically the last
navigational page and is referenced from everywhere:
```markdown
← Previous: [Home](https://github.com/<org>/<repo>/wiki) | Next: — →
```

---

## Heading structure with emojis

Emojis are **mandatory** on every page of the wiki, including `Home.md`, `README.md`, and
all content pages. A wiki page without emojis at the correct heading levels is incomplete.

### Rules

- **H1 (`#`)**: always one per page, always has an emoji prefix.
- **H2 (`##`)**: always has an emoji prefix.
- **H3 (`###`) numbered steps**: emojis are allowed and expected when the H3 is a numbered
  step inside a Get-Started or step-by-step section (e.g., `### 📥 1. Install`). The emoji
  must relate semantically to the step action, not just decorate.
- **H3 (`###`) reference or detail sections**: no emojis (e.g., `### Available flags`).
- **H4 and below**: no emojis.
- One emoji per heading (never two).
- Emoji precedes the title text with a single space: `## 🚀 Quick Start`.

### Standard Kozen emoji map

Use these consistently across all wikis for semantic coherence. Choosing the wrong emoji for
a section (e.g., 📦 for a Security section) undermines the semantic value.

| Emoji | Semantic | Typical heading |
|---|---|---|
| 🚀 | Getting started, launching, demos | `# 🚀 Kozen Trigger`, `## 🚀 Get Started`, `## 🚀 Simple Demo: Step by Step` |
| 📥 | Install, download, pull | `### 📥 1. Install` |
| 🤖 | AI, automation, MCP, delegate creation | `### 🤖 2. Create the delegate`, `## 🤖 MCP Integration` |
| 📝 | Configuration files, notes, env files | `### 📝 3. Create the env file` |
| 🧑‍💻 | Starting a service, developer action | `### 🧑‍💻 4. Start the service` |
| 🔍 | Observe output, verify, debug, logs | `### 🔍 5. Observe logs` |
| 🔧 | Configuration, setup, settings | `## 🔧 Configuration` |
| 📚 | Documentation, references, further reading | `## 📚 References` |
| 🔐 | Security, encryption, authentication | `## 🔐 Security` |
| ⚙️ | Options, parameters, environment variables | `## ⚙️ Environment Variables` |
| ✅ | Best practices, requirements met | `## ✅ Best Practices` |
| ❌ | Errors, what to avoid, missing | `## ❌ Troubleshooting` |
| ⚡ | Key features, highlights, performance | `## ⚡ Key Features` |
| 🎯 | Purpose, goals, why this matters | `## 🎯 Why Use This?` |
| 🌟 | Key concepts, highlights, overview | `## 🌟 Overview` |
| 🏭 | Architecture, systems, components | `## 🏭 Architecture` |
| 📦 | Packages, npm, installation | `## 📦 Installation` |
| 🔑 | API keys, credentials, master key | `## 🔑 Authentication` |
| ⚖️ | Legal, policy, licence, disclaimer | `# ⚖️ Disclaimer and Usage Policy` |
| 🧠 | Delegate, business logic, smart processing | `## 🧠 Delegate` |
| 🛡️ | IAM, compliance, least privilege | `## 🛡️ IAM Rectification` |
| 🗺️ | Roadmap, future features, coming soon | `## 🗺️ Roadmap` |

---

## Module display name: human-friendly, not the npm package name

The npm package name (`@kozen/etl-mk`, `@kozen/trigger`) is a technical identifier for
install commands and `import` statements. It is **never** the display name in headings,
introductions, or prose.

Derive the human-friendly display name by expanding the abbreviations and adding context:

| npm package name | Display name |
|---|---|
| `@kozen/engine` | Kozen Engine |
| `@kozen/trigger` | Kozen Trigger |
| `@kozen/secret` | Kozen Secret |
| `@kozen/iam-rectification` | Kozen IAM Rectification |
| `@kozen/etl-mk` | Kozen Module: ETL MongoDB - Kafka |

**Rules for constructing a display name:**

1. Always prefix with **Kozen**: it anchors the module in the ecosystem.
2. Expand all abbreviations to their full English words (`etl` → `ETL`, `mk` → `MongoDB - Kafka`,
   `iam` → `IAM`, `mcp` → `MCP`).
3. If the abbreviation implies a pipeline or connection between two systems, separate the
   systems with a hyphen: `ETL MongoDB - Kafka`, not `ETL MongoDB Kafka`.
4. Use title case.
5. If the module name is unclear or ambiguous, ask the user before inventing a display name.

**Where to use the display name:**

- H1 heading: `# 🚀 Kozen Trigger: MongoDB Change Stream Event Handler`
- Introduction sentences: "Kozen Trigger is a Kozen module that…"
- References and links in other wiki pages: `[Kozen Secret](url)`

**Where to use the npm package name:**

- `npm install` commands
- `import` / `require` statements
- `KOZEN_MODULE_LOAD` env var values
- Code comments referencing the package

---

## GitHub wiki URL pattern for Kozen modules

All Kozen module repositories live under the `https://github.com/kozen-labs/` organization
by default. Do not reference other GitHub organizations or repositories unless the user
explicitly states a different location.

The base URL for any Kozen module wiki follows this pattern:

```
https://github.com/kozen-labs/<module-repo-name>/wiki/<Page-Name>
```

| npm package | Repo name | Base wiki URL |
|---|---|---|
| `@kozen/engine` | `engine` | `https://github.com/kozen-labs/engine/wiki` |
| `@kozen/trigger` | `trigger` | `https://github.com/kozen-labs/trigger/wiki` |
| `@kozen/secret` | `secret` | `https://github.com/kozen-labs/secret/wiki` |
| `@kozen/iam-rectification` | `iam-rectification` | `https://github.com/kozen-labs/iam-rectification/wiki` |
| `@kozen/etl-mk` | `etl-mk` | `https://github.com/kozen-labs/etl-mk/wiki` |

Page URL rules:
- Page name = file name without `.md` extension.
- Multi-word page names use PascalCase + hyphens: `Get-Started`, `App-CLI`, `Kozen-Integration`.
- The Home page URL ends at `/wiki`, not `/wiki/Home`.
- Example: `Delegate` page for `etl-mk` → `https://github.com/kozen-labs/etl-mk/wiki/Delegate`

---

## The `**Note:**` pattern for contextual explanations

After code blocks in Get-Started and CLI pages, add a `**Note:**` paragraph to explain
context, caveats, or alternatives that the reader would otherwise have to discover by trial
and error. This is a first-class documentation pattern in Kozen wikis.

### Format

```markdown
**Note:** [One or two sentences. Explain why the above is the way it is, what alternatives
exist, or what happens if the reader does not follow the instruction. Do not repeat what the
code already shows.]
```

### When to use it

- After an install command: clarify global vs local install implications.
- After a `.env` file example: clarify that system environment variables eliminate the need
  for the file.
- After the `--envFile` flag: clarify it is optional when env vars are already set.
- After a log output block: clarify which component produced the output and how to adjust it.
- After a delegate code example: clarify which exports are required vs optional.

### Example (from existing Kozen wikis)

````markdown
```shell
npx kozen --action=trigger:start --envFile=/home/user/.env
```

**Note:** Include `--envFile=/home/user/.env` only if you want to load local environment
variables. If all variables are already set in the global environment, this option is not
required.
````

Do not use `**Note:**` for information the reader strictly needs to complete the step. That
belongs in the body text before the code block, not after it.

---

## Markdown-first: avoid HTML

All wiki and README content must be written in standard Markdown. HTML is permitted only
when Markdown cannot produce the required result and the user has explicitly requested it,
or when there is a clear technical necessity with no Markdown equivalent.

| Situation | Use Markdown | Use HTML only if |
|---|---|---|
| Headings | `## Heading` | — never use `<h2>` |
| Bold / italic | `**bold**`, `*italic*` | — never use `<strong>`, `<em>` |
| Links | `[text](url)` | — never use `<a href>` |
| Images | `![alt](url)` | — never use `<img>` |
| Tables | GFM pipe tables | the table needs `colspan`/`rowspan` (rare) |
| Line breaks | Blank line between paragraphs | `<br>` only when a forced mid-paragraph break is genuinely necessary |
| Collapsible sections | — | `<details><summary>` is acceptable when content is truly optional and long |
| Badges / shields | `![badge](url)` image syntax | — |

**Rules:**

- Never wrap content in `<div>`, `<span>`, or `<p>` tags. Markdown renderers handle this automatically.
- GitHub Wiki and most npm/PyPI renderers handle GitHub-Flavored Markdown (GFM) natively;
  HTML inside Markdown is rendered inconsistently across platforms.
- If you are tempted to add HTML for styling (color, font size, alignment): do not. Use
  Markdown structure (headings, bold, tables, blockquotes) to convey emphasis instead.

---

## File naming

| Convention | Correct | Incorrect |
|---|---|---|
| PascalCase + hyphens | `Get-Started.md` | `get-started.md`, `GetStarted.md` |
| Multi-word acronyms | `App-CLI.md`, `App-MCP.md` | `AppCLI.md`, `app-cli.md` |
| Home is always `Home.md` | `Home.md` | `home.md`, `index.md` |
| Policy is always `POLICY.md` | `POLICY.md` | `policy.md`, `Policy.md` |
| Descriptive names | `Kozen-Module-Development.md` | `Module.md`, `Dev.md` |

---

## Link format

### Internal wiki links (absolute, mandatory)

Always use the full GitHub wiki URL. Never use relative paths for GitHub Wiki mode.

```markdown
[Get Started](https://github.com/<org>/<repo>/wiki/Get-Started)
[Home](https://github.com/<org>/<repo>/wiki)
[POLICY](https://github.com/<org>/<repo>/wiki/POLICY)
```

The Home page URL ends at `/wiki`, not `/wiki/Home`.

### Cross-wiki links (linking from one module wiki to another)

```markdown
For the IoC container reference, see the [Kozen Engine wiki](https://github.com/kozen-labs/engine/wiki).
```

### External links

Use descriptive anchor text. Never use "click here" or bare URLs.

```markdown
[MongoDB Change Streams documentation](https://www.mongodb.com/docs/manual/changeStreams/)
```

---

## Code blocks

Always specify the language identifier:

| Language | Tag |
|---|---|
| TypeScript | `typescript` or `ts` |
| JavaScript | `javascript` or `js` |
| Bash / Shell | `bash` |
| PowerShell | `powershell` |
| JSON | `json` |
| YAML | `yaml` |
| Markdown | `markdown` |

Label code files with a comment when the snippet comes from a specific file:

```typescript
// src/configs/ioc.json excerpt
```

Show realistic values. Never use `foo`, `bar`, or `baz` as placeholder names in Kozen wikis.
Use `mydb`, `orders`, `appUser`, `MY_API_KEY` etc.

---

## Tables

Prefer tables over bulleted lists for any content with three or more structured attributes:

```markdown
| Flag | Env var | Required | Description |
|---|---|---|---|
| `--uri` | `KOZEN_TRIGGER_URI` | Yes | MongoDB connection string |
```

Use `—` in a table cell to indicate "not applicable" or "none".

---

## Tone and voice

Follow the combined principles of the `technical-writing` and `ks-technical-article-writer` skills:

- Write in English.
- Address the reader as a peer developer: assume competence, do not over-explain basics.
- Active voice: "The module connects to MongoDB" not "MongoDB is connected to by the module."
- One idea per paragraph.
- Sentences under 25 words for complex technical content.
- No filler phrases: "In conclusion", "It is worth noting that", "As mentioned earlier."
- Expand acronyms on first use: "Model Context Protocol (MCP)".
- No fabricated claims: every technical statement must be verifiable from source code.
- Motivational without being salesy: explain what the reader gains, not what the product does.

---

## Punctuation: avoid the em dash in prose

The em dash (`—`) must **not** be used to separate, highlight, or connect ideas in body
text. It makes sentences harder to scan and is not idiomatic in technical writing.

**Permitted uses of `—` (two cases only):**
- Navigation footer placeholder when no previous or next page exists: `← Previous: —`
- Table cells to mean "not applicable" or "none"

**Replace every other use with a common alternative:**

| Instead of `—` | Use | Example |
|---|---|---|
| Connector between clause and explanation | `:` | "The module reads config: URI, database, collection." |
| Parenthetical aside | commas or `()` | "The service (registered as singleton) handles requests." |
| "that is" or "which is" | rewrite the sentence | "The token `logger:service`, which is a singleton, ..." |
| Emphasis or highlight | `**bold**` or sentence break | Start a new sentence instead of bolting it onto the previous one. |
| List introductions | `:` | "Three options are available:" |

**Incorrect:**
```
The module connects to MongoDB — using SCRAM or X.509.
```

**Correct:**
```
The module connects to MongoDB using SCRAM or X.509.
The module supports two authentication methods: SCRAM and X.509.
```

This rule applies to all content: headings, bullet points, table cells (except N/A), body
paragraphs, notes, and code comments inside wiki pages.

---

## Page ordering within a module wiki

The standard page order for any Kozen module wiki:

1. `Home.md`: overview, features, quick install, links to all other pages, references, disclaimer link
2. `Get-Started.md`: step-by-step first use (install, configure, run)
3. `Configuration.md` or `Config.md`: all options, env vars, config files
4. `[Interface]-via-CLI.md`: CLI actions and flags (when module has CLI)
5. `[Interface]-via-MCP.md`: MCP tools and schemas (when module has MCP)
6. `Delegate.md` or `API.md`: handler or programmatic API reference
7. `Kozen-Integration.md`: how to use as a Kozen IoC module (when applicable)
8. `POLICY.md`: licence and disclaimer (always last in the ordered list)

Not every module needs all pages. `@kozen/trigger` has no MCP page. `@kozen/engine` has
both `App-CLI.md` and `App-MCP.md`. Omit pages that have no content for the module.

---

## Standard References section format

Use the following format at the bottom of every page that cites external resources:

```markdown
## 📚 References

- [Kozen Engine Wiki](https://github.com/kozen-labs/engine/wiki): core framework documentation
- [MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/): official MongoDB documentation
- [Disclaimer and Usage Policy](https://github.com/<org>/<repo>/wiki/POLICY): usage terms and licence
```

Home pages include a richer references block that also links to:
- DeepWiki (if available)
- GitHub repository
- npm package
- Issues tracker
- Contributing guidelines

---

## Disclaimer link (mandatory on Home.md)

Every `Home.md` must include a visible reference to the POLICY page:

```markdown
> This project is open source and distributed under the terms described in the
> [Disclaimer and Usage Policy](https://github.com/<org>/<repo>/wiki/POLICY).
```

This can appear in the References section or as a standalone blockquote near the bottom.
