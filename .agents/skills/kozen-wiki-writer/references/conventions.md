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
- Use `—` (em dash, not a hyphen) when there is no previous or next page.
- The em dash `—` is the only place in wiki body text where it is permitted as a separator.
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

### Rules
- H1 (`#`) — always one per page, always has an emoji prefix
- H2 (`##`) — use emojis; maximum **two emojis visible on one screen** at a time
- H3 (`###`) and below — **no emojis**
- One emoji per heading (not two)
- Emoji precedes the title text with a space: `## 🚀 Quick Start`

### Standard Kozen emoji map

Use these consistently across all wikis for semantic coherence:

| Emoji | Semantic | Typical section title |
|---|---|---|
| 🚀 | Getting started, launching, speed | Quick Start, Installation, Get Started |
| 🔧 | Configuration, setup, tools | Configuration, Setup, Settings |
| 📚 | Documentation, references, learning | References, Documentation, Further Reading |
| 🔐 | Security, encryption, authentication | Security, Encryption, Authentication |
| ⚙️ | Settings, parameters, options | Options, Parameters, Environment Variables |
| 🤖 | AI, automation, MCP | MCP Integration, AI Tools, Automation |
| ✅ | Success, validation, correct | Best Practices, Passing, Requirements Met |
| ❌ | Error, missing, failure | Missing Permissions, Errors, What to Avoid |
| ⚡ | Key features, important, fast | Key Features, Why Use This, Highlights |
| 🎯 | Goals, objectives, purpose | Purpose, Objectives, Why This Matters |
| 📝 | Notes, examples, code | Examples, Notes, Code Samples |
| 🌟 | Highlights, key concepts | Key Concepts, Highlights, Core Ideas |
| 🏭 | Architecture, systems, modules | Architecture, Module System, Components |
| 📦 | Packages, modules, installation | Installation, Packages, npm |
| 🔑 | Keys, access, credentials | API Keys, Authentication, Master Key |
| ⚖️ | Legal, policy, licence | Disclaimer, Policy, Legal Notice |
| 🧠 | Logic, delegates, smart systems | Delegate, Business Logic, Processing |
| 🛡️ | Protection, compliance, IAM | IAM, Compliance, Least Privilege |

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
- No fabricated claims — every technical statement must be verifiable from source code.
- Motivational without being salesy: explain what the reader gains, not what the product does.

---

## Page ordering within a module wiki

The standard page order for any Kozen module wiki:

1. `Home.md` — overview, features, quick install, links to all other pages, references, disclaimer link
2. `Get-Started.md` — step-by-step first use (install, configure, run)
3. `Configuration.md` or `Config.md` — all options, env vars, config files
4. `[Interface]-via-CLI.md` — CLI actions and flags (when module has CLI)
5. `[Interface]-via-MCP.md` — MCP tools and schemas (when module has MCP)
6. `Delegate.md` or `API.md` — handler or programmatic API reference
7. `Kozen-Integration.md` — how to use as a Kozen IoC module (when applicable)
8. `POLICY.md` — licence and disclaimer (always last in the ordered list)

Not every module needs all pages. `@kozen/trigger` has no MCP page. `@kozen/engine` has
both `App-CLI.md` and `App-MCP.md`. Omit pages that have no content for the module.

---

## Standard References section format

Use the following format at the bottom of every page that cites external resources:

```markdown
## 📚 References

- [Kozen Engine Wiki](https://github.com/kozen-labs/engine/wiki) — framework documentation
- [MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/) — official driver documentation
- [Disclaimer and Usage Policy](https://github.com/<org>/<repo>/wiki/POLICY) — terms and licence
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
