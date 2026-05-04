---
name: kozen-wiki-writer
description: >
  Expert guide for producing Kozen module wiki documentation. Covers the full authoring
  workflow: choosing a publication mode (GitHub Wiki vs docs/ vs tmp/wiki), analysing a
  module's source code to identify what must be documented (entry point exports, CLI help
  files, IoC configs, package.json), structuring the page set with escalated progressive
  content, writing each page type (Home, Get-Started, Configuration, API, CLI, MCP, Delegate,
  POLICY), applying the exact navigation footer, emoji-heading, and link conventions used
  across all Kozen wikis, enforcing the no-duplication rule, adding bibliography sections and
  licence/disclaimer pages, and using the technical-writing and ks-technical-article-writer
  standards throughout.
created: 2026-05-04
updated: 2026-05-04
---

# Kozen Wiki Writer

This skill produces wiki documentation for Kozen modules (`@kozen/engine`, `@kozen/secret`,
`@kozen/trigger`, `@kozen/iam-rectification`, and any future module). The output matches the
conventions established across the four existing wikis (`engine.wiki`, `secret.wiki`,
`trigger.wiki`, `iam-rectification.wiki`).

---

## Routing table

| Signal | Reference |
|---|---|
| publication mode, GitHub wiki, docs folder, tmp/wiki, where to put wiki files, git wiki repo, github pages, docs vs wiki, ask user publication, publish kozen wiki | `references/publication-modes.md` |
| file naming, navigation footer, previous next links, emoji heading, heading structure, link format, absolute URL wiki, code block convention, table convention, tone voice, emoji mapping, page order, PascalCase hyphen | `references/conventions.md` |
| what to document, page types, Home page, Get-Started page, Configuration page, API page, CLI page, MCP page, Delegate page, POLICY page, licence page, disclaimer page, escalated content, progressive disclosure, no duplication rule, bibliography section, references section, entry point analysis, exported classes wiki, src/index.ts wiki, src/docs wiki, package.json wiki, IoC config wiki, do not repeat | `references/content-structure.md` |

---

## When to use this skill

- Writing or rewriting a wiki for any Kozen module from scratch
- Adding a new page to an existing Kozen wiki
- Reviewing an existing wiki page for convention compliance
- Deciding where to store wiki files before publishing them
- Understanding what source files to read before writing any documentation

---

## Quick start

Ask the user two questions before writing anything:

1. **Publication mode** — "Do you want to publish as a GitHub Wiki, as a `docs/` site, or
   generate files in `tmp/wiki/` for later decision? See `references/publication-modes.md`
   for the full trade-off table."

2. **Disclaimer preference** — "Should every page include a link to the POLICY/Disclaimer
   page, or should the disclaimer be in the Home page only?"

Once answered, analyse the module source code (see `references/content-structure.md`,
section *Source analysis checklist*), plan the page set and order, then write page by page.

---

## Do & Don't

**Do:**
- Always write the POLICY (disclaimer/licence) page first — it is referenced by every Home page.
- Always write Home.md last — it links to all other pages and summarises the module.
- Use the exact navigation footer format from `references/conventions.md`.
- Add emojis only to H1 and H2 headings — one per heading, maximum two per screen.
- Use absolute GitHub wiki URLs for all internal links.
- Add a **References** or **Bibliography** section to every page that cites external resources.
- Keep licence notice and disclaimer in POLICY.md; link to it from Home.md and Get-Started.md.
- Use progressive disclosure: each page assumes the reader has read all previous pages.
- Read `src/index.ts` to find every exported class, interface, and type before writing the API page.
- Read `src/docs/*.txt` to reuse the authoritative CLI help text.

**Don't:**
- Never duplicate content — if something is explained on one page, link to it from the next.
- Never use relative links for internal wiki pages (always full GitHub URLs).
- Never invent API signatures — read the source code first.
- Never skip the navigation footer, even on the last page (Next becomes `—`).
- Never omit the POLICY page — it protects the authors legally.
- Never add emojis to H3 or lower headings.
- Never use em dashes (`—`) as connectors in body text (only in the navigation footer and tables).
