---
name: kozen-wiki-writer
description: >
  Expert guide for producing Kozen module project documentation: GitHub wiki pages, README
  synchronisation, and GitHub Issues. Covers the full authoring workflow: choosing a
  publication mode (GitHub Wiki vs docs/ vs tmp/ staging), the Home.md = README.md parity
  rule (minimal high-level content that works on GitHub, npm, and PyPI), progressive
  disclosure across the page set, source analysis checklist (entry point exports, CLI help
  files, IoC configs, package.json), all format conventions (navigation footer, emoji-heading
  map, PascalCase-hyphen naming, absolute URLs), the no-duplication rule, bibliography and
  licence/disclaimer structure, and standardised GitHub Issues (title, problem description,
  location, consequences, reproduction steps, possible solutions) with publication modes for
  GitHub-managed or docs-managed issue tracking.
created: 2026-05-04
updated: 2026-05-04
---

# Kozen Wiki Writer

This skill produces all documentation artifacts for Kozen modules: wiki pages, README files,
and GitHub Issues. Output matches the conventions established across the four existing wikis
(`engine.wiki`, `secret.wiki`, `trigger.wiki`, `iam-rectification.wiki`).

---

## Scope: wiki + issues in one skill

Wiki documentation and GitHub Issues are kept in the same skill because they share the same
publication-mode decision, the same staging directory (`tmp/`), and the same "ask the user
first" pattern. Splitting them would force users to load two skills for the same project
documentation task. If a future team needs a generic, Kozen-agnostic issue template skill,
the issue section of `references/issues.md` can be extracted then.

---

## Routing table

| Signal | Reference |
|---|---|
| publication mode, GitHub wiki, docs folder, tmp/wiki, where to put wiki files, git wiki repo, github pages, docs vs wiki, ask user publication, publish kozen wiki | `references/publication-modes.md` |
| file naming, navigation footer, previous next links, emoji heading, heading structure, emoji on H3 numbered steps, link format, absolute URL wiki, code block convention, table convention, tone voice, emoji mapping, emoji map, semantic emoji, page order, PascalCase hyphen, Note pattern, Note block after code, contextual note, kozen-labs URL, wiki URL pattern, GitHub wiki URL, module display name, human-friendly name, npm package name vs display name, expand abbreviations, Markdown first, avoid HTML, no HTML tags, GFM, GitHub Flavored Markdown | `references/conventions.md` |
| what to document, page types, Home page minimal, Home README parity, README.md same as Home, npm readme, pypi readme, SEO Home, SEO optimization wiki, incite exploration, problem-solution framing, discoverability, H1 module name, first paragraph lede, Get-Started page, Get-Started numbered steps, step by step template, Get-Started emoji steps, Configuration page, API page, CLI page, MCP page, Delegate page, POLICY page, licence page, disclaimer page, escalated content, progressive disclosure, no duplication rule, bibliography section, references section, entry point analysis, exported classes wiki, src/index.ts wiki, source analysis checklist, what to read before writing, practical code examples, runnable snippet, example per page, when to include examples, complete example, realistic example, TypeScript example | `references/content-structure.md` |
| GitHub Issues, issue template, create issue, write issue, issue structure, issue title, issue description, bug report, feature request, issue location, consequences, reproduction steps, possible solutions, tmp/issues, docs/issues, issue publication mode, standardised issues, project management docs, source audit findings, code review findings, classify findings, identify problems, detect issues, improve module | `references/issues.md` |

---

## When to use this skill

- Writing or rewriting a wiki for any Kozen module from scratch
- Writing or reviewing a module README (must match Home.md)
- Adding a new page to an existing Kozen wiki
- Reviewing a page for convention compliance
- **Auditing source code to identify improvements and issues** — reading the module source,
  detecting problems, gaps, or opportunities, and converting findings into standardised
  GitHub Issues before writing or updating documentation
- Creating GitHub Issues in the project's standardised format
- Deciding whether to manage issues via GitHub or in a `docs/` directory

---

## Source audit workflow (identify issues before writing)

Before producing any wiki page or issue file, read the module source code. The source audit
serves two purposes at once: it supplies the verified facts that go into the documentation,
and it surfaces problems that should become GitHub Issues.

**Step 1 — Read the source** (same checklist as `references/content-structure.md`):

| File | What to look for |
|---|---|
| `src/index.ts` | Missing exports, incorrect types, undocumented public API |
| `src/docs/*.txt` | Outdated flag names, missing env vars, wrong default values |
| `src/configs/ioc.json` | Tokens registered but never documented; incorrect lifetimes |
| `src/configs/cli.json` | Controllers that lack a CLI help file |
| `src/configs/mcp.json` | MCP tools without input schema validation |
| `src/services/*.ts` | Methods with no error handling at system boundaries |
| `src/controllers/*.ts` | Actions not documented in the help file |
| `package.json` | Licence, version, homepage — verify they match the wiki |

**Step 2 — Classify each finding** into one of these categories:

| Category | Creates issue type | Example |
|---|---|---|
| Incorrect behaviour or data loss | `bug` | Change stream closes on empty collection |
| Missing capability a user would expect | `feat` | No retry on transient MongoDB disconnection |
| Documentation absent or misleading | `docs` | X.509 certificate path not documented |
| Internal quality or maintainability | `refactor` or `chore` | Service class has no unit tests |
| Credential or data exposure | `security` | Master key appears in debug log |

**Step 3 — Draft issue files** following `references/issues.md` for every finding, then
present them to the user for review before writing any wiki page. This ensures the wiki
documents the corrected state of the module, not the current (possibly broken) state.

**Step 4 — Write documentation** only after the user has reviewed and accepted or dismissed
the findings. Document the intended correct behaviour, not workarounds for open bugs.

---

## Questions to ask before starting

Ask all three questions before generating any file. Defaults are shown in parentheses.

**1. Publication mode for wiki pages**
> A) GitHub Wiki — files in `tmp/wiki/`, later pushed to the `*.wiki.git` repo
> B) `docs/` directory — files versioned alongside source code
> C) `tmp/wiki/` staging — undecided; generate with GitHub Wiki conventions

**2. Disclaimer preference**
> A) POLICY.md linked from every page's References section
> B) Disclaimer link only in Home.md (default)
> C) No disclaimer — user explicitly opts out

**3. Issue management mode** (only if the user mentions issues)
> A) GitHub Issues — files in `tmp/issues/`, published via GitHub web UI or CLI
> B) Manual docs — files in `docs/issues/` or `doc/issues/`, versioned with source code
> C) Not needed for this project

---

## Do & Don't

**Do:**
- Audit the source code first — read `src/index.ts`, `src/docs/*.txt`, and `src/configs/*.json` before writing a single line of documentation.
- Convert every finding from the audit into a standardised issue file (see `references/issues.md`) and present it to the user before writing wiki pages.
- Include practical code examples on every page that requires them (see the per-page table in `references/content-structure.md`). Examples must be complete, runnable, and use realistic values.
- Write POLICY.md first — every Home.md links to it.
- Write Home.md last — it links to every other page.
- Keep Home.md minimal: high-level overview, key features, one install command, links. No API details.
- Mirror Home.md content in README.md so the module is navigable from GitHub, npm, and PyPI.
- Add emojis to H1 and H2 headings on **every** page — Home, README, and all wiki pages. No page is exempt.
- Add emojis to H3 headings in Get-Started numbered steps (e.g., `### 📥 1. Install`). Use semantically relevant emojis from the map in `references/conventions.md`.
- Use the exact navigation footer format from `references/conventions.md` on every page.
- Use `**Note:**` blocks after code blocks to explain context, caveats, and alternatives (see `references/conventions.md`).
- Write Home.md H1 as `# [emoji] [Module Name] — [one-line function]` for SEO discoverability.
- Write the first paragraph of Home.md as a problem-solution statement that incites the reader to explore further.
- Use the human-friendly display name in all headings and prose — never the npm package name (e.g., `Kozen Trigger`, not `@kozen/trigger`). Reserve the npm name for `npm install` commands, `import` statements, and `KOZEN_MODULE_LOAD` values.
- Default all repository and wiki URLs to `https://github.com/kozen-labs/` unless the user explicitly specifies a different organization.
- Write everything in Markdown (GFM). Use HTML only when Markdown has no equivalent and the user requests it, or for `<details>/<summary>` collapsible sections.
- Use absolute GitHub wiki URLs (`https://github.com/kozen-labs/<module>/wiki/<Page>`) for all internal wiki links.
- Add a References section to every page that cites external resources.
- Verify every API signature against `src/index.ts` before writing.
- Verify every CLI action and flag against `src/docs/*.txt` before writing.

**Don't:**
- Never put API details, flag tables, or configuration options on Home.md — link to the dedicated page.
- Never duplicate content — if explained on one page, link to it from all others.
- Never use relative links for internal wiki pages (GitHub Wiki does not resolve them).
- Never invent API signatures — read the source first.
- Never skip the navigation footer, even on the last page.
- Never omit POLICY.md — it is a legal requirement for open source modules.
- Never add emojis to H3 or lower headings except for numbered steps in Get-Started sections.
- Never write a Home.md H1 as just `# Home` or `# [Module Name]` without a function description — it harms SEO.
- Never use the npm package name (`@kozen/etl-mk`) as the display name in headings or prose — derive a human-friendly name and confirm with the user if ambiguous.
- Never reference a GitHub organization other than `kozen-labs` unless the user explicitly provides a different URL.
- Never use HTML tags (`<div>`, `<span>`, `<p>`, `<h2>`, `<a>`, etc.) when Markdown syntax exists for the same purpose.
- Never use em dashes (`—`) as body-text connectors (only in the navigation footer and table cells).
- Never write a `**Note:**` that repeats information already in the code block or the sentence immediately before it.
