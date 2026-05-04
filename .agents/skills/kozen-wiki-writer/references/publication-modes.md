---
name: Kozen Wiki — Publication Modes
description: >
  Deciding where to generate and store Kozen wiki files: GitHub Wiki (*.wiki repo), docs/
  directory (GitHub Pages / static site), or tmp/wiki/ (intermediate staging). Covers the
  trade-offs of each mode, the exact questions to ask the user, output directory conventions,
  file naming rules per mode, and Git workflow notes.
category: kozen-wiki-writer
tags:
  - wiki
  - publication
  - github-wiki
  - docs
  - github-pages
---

# Kozen Wiki — Publication Modes

Before writing a single line of documentation, ask the user which publication mode they
want. The choice affects file locations, link formats, and the Git workflow.

---

## The three modes

### Mode A — GitHub Wiki (`*.wiki` repo)

GitHub Wikis are stored in a separate Git repository automatically created alongside every
GitHub repository. The clone URL is `https://github.com/<org>/<repo>.wiki.git`.

**File location:** Generate files in `tmp/wiki/` (staging area). After review, the user
clones the wiki repo and copies the files there.

**Directory layout:**
```
tmp/wiki/
├── Home.md
├── Get-Started.md
├── Configuration.md
├── API.md
├── Delegate.md          ← trigger only
├── CLI.md               ← modules with CLI actions
├── MCP.md               ← modules with MCP tools
└── POLICY.md
```

**File naming rules:**
- PascalCase with hyphens for multi-word names: `Get-Started.md`, `App-CLI.md`, `IAM-Util-Package.md`
- No underscores, no spaces, no lowercase-only names
- The GitHub wiki renders the filename as the page title (hyphens become spaces)

**Link format:** All internal links use the full absolute URL:
```
https://github.com/<org>/<repo>/wiki/<PageName>
```
Example:
```
[Get Started](https://github.com/kozen-labs/trigger/wiki/Get-Started)
```
For the Home page specifically:
```
[Home](https://github.com/kozen-labs/trigger/wiki)
```
Do NOT use relative paths like `./Get-Started.md` — GitHub wiki does not resolve them.

**Pros:**
- Separated from source code — contributors can edit docs without a PR
- GitHub provides a built-in table of contents sidebar
- Zero deployment configuration

**Cons:**
- Not version-controlled alongside the source code (separate repo)
- No pull request workflow by default (though you can enable it)
- Limited styling — plain GitHub markdown only

**Recommended for:** Public-facing module documentation that should be editable by
non-developers and browseable on GitHub without any setup.

---

### Mode B — `docs/` directory (GitHub Pages / static site)

Documentation lives inside the module repository under a `docs/` folder. GitHub Pages
can serve it directly, or a static site generator (Docusaurus, MkDocs, VitePress) can
process it.

**File location:** Generate files directly in `docs/` at the project root.

**Directory layout:**
```
docs/
├── index.md             ← equivalent to Home.md
├── get-started.md
├── configuration.md
├── api.md
├── delegate.md
├── cli.md
├── mcp.md
└── policy.md
```

**File naming rules:**
- All lowercase, hyphens for spaces: `get-started.md`, `app-cli.md`
- Consistent with static site generator conventions

**Link format:** Relative paths are acceptable (and preferred) in this mode:
```
[Get Started](./get-started.md)
```
Or root-relative if the site has a known base URL:
```
[Get Started](/get-started)
```

**Pros:**
- Version-controlled alongside source code — changes to code and docs go in the same PR
- Works with static site generators for rich styling, search, and versioning
- Can be reviewed via pull requests

**Cons:**
- Requires GitHub Pages or a CI/CD deployment pipeline
- More setup than a wiki

**Recommended for:** Projects that want docs versioned with code and that have or plan to
set up GitHub Pages or a static site.

---

### Mode C — `tmp/wiki/` (staging / undecided)

The user has not decided on a final publication target. Generate files in `tmp/wiki/` so
they can be reviewed and moved to Mode A or Mode B later.

**File location:** `tmp/wiki/` in the project root.

**File naming:** Follow GitHub Wiki conventions (PascalCase + hyphens, Mode A). This makes
it easy to copy to a wiki repo later without renaming.

**Link format:** Use full GitHub Wiki URLs (Mode A format) as placeholders, since the
organization and repo name are known even if the wiki has not been created yet.

**Recommendation to give the user:**

> The files are in `tmp/wiki/` using GitHub Wiki conventions. When you are ready to publish:
>
> **Option A — GitHub Wiki:**
> ```bash
> git clone https://github.com/<org>/<repo>.wiki.git wiki-repo
> cp tmp/wiki/*.md wiki-repo/
> cd wiki-repo && git add . && git commit -m "docs: initial wiki" && git push
> ```
>
> **Option B — docs/ folder:**
> Rename files to lowercase, update relative links, move to `docs/`.

---

## Questions to ask the user

Ask both questions before generating any file:

```
1. Publication mode:
   A) GitHub Wiki (files in tmp/wiki/, later pushed to the *.wiki.git repo)
   B) docs/ directory (files in docs/, versioned with source code)
   C) tmp/wiki/ staging (undecided — generate with GitHub Wiki conventions for flexibility)

2. Disclaimer preference:
   A) POLICY.md linked from every page footer
   B) Disclaimer text only in Home.md, not on every page
   C) No disclaimer (not recommended for public modules)
```

---

## Per-mode summary table

| Aspect | GitHub Wiki | docs/ | tmp/wiki staging |
|---|---|---|---|
| Output directory | `tmp/wiki/` → push to `*.wiki.git` | `docs/` | `tmp/wiki/` |
| File naming | PascalCase-Hyphen.md | lowercase-hyphen.md | PascalCase-Hyphen.md |
| Internal links | Absolute GitHub wiki URLs | Relative paths | Absolute GitHub wiki URLs |
| Version-controlled with code | No (separate repo) | Yes | Staging only |
| Pull request review | Not by default | Yes | N/A |
| Deployment required | No | Yes (GitHub Pages / CI) | N/A |
| Best for | Public docs, non-dev editors | Versioned docs, static sites | Undecided projects |

---

## Git workflow for GitHub Wiki (Mode A)

```bash
# Clone the wiki repo (created automatically by GitHub when you enable the wiki)
git clone https://github.com/<org>/<module-name>.wiki.git tmp/wiki-publish

# Copy generated files
cp tmp/wiki/*.md tmp/wiki-publish/

# Commit and push
cd tmp/wiki-publish
git add .
git commit -m "docs: add initial wiki pages"
git push origin master
```

The wiki repo always uses the `master` branch (not `main`).
