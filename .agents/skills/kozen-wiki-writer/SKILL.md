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
| file naming, navigation footer, previous next links, emoji heading, heading structure, link format, absolute URL wiki, code block convention, table convention, tone voice, emoji mapping, page order, PascalCase hyphen | `references/conventions.md` |
| what to document, page types, Home page minimal, Home README parity, README.md same as Home, npm readme, pypi readme, Get-Started page, Configuration page, API page, CLI page, MCP page, Delegate page, POLICY page, licence page, disclaimer page, escalated content, progressive disclosure, no duplication rule, bibliography section, references section, entry point analysis, exported classes wiki, src/index.ts wiki | `references/content-structure.md` |
| GitHub Issues, issue template, create issue, write issue, issue structure, issue title, issue description, bug report, feature request, issue location, consequences, reproduction steps, possible solutions, tmp/issues, docs/issues, issue publication mode, standardised issues, project management docs | `references/issues.md` |

---

## When to use this skill

- Writing or rewriting a wiki for any Kozen module from scratch
- Writing or reviewing a module README (must match Home.md)
- Adding a new page to an existing Kozen wiki
- Reviewing a page for convention compliance
- Creating GitHub Issues in the project's standardised format
- Deciding whether to manage issues via GitHub or in a `docs/` directory

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
- Write POLICY.md first — every Home.md links to it.
- Write Home.md last — it links to every other page.
- Keep Home.md minimal: high-level overview, key features, one install command, links. No API details.
- Mirror Home.md content in README.md so the module is navigable from GitHub, npm, and PyPI.
- Use the exact navigation footer format from `references/conventions.md`.
- Add emojis only to H1 and H2 headings.
- Use absolute GitHub wiki URLs for all internal wiki links.
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
- Never add emojis to H3 or lower headings.
- Never use em dashes (`—`) as body-text connectors (only in the navigation footer and table cells).
