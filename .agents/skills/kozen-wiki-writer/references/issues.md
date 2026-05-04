---
name: Kozen Wiki — GitHub Issues Management
description: >
  Standardised GitHub Issues structure for Kozen module projects: issue file format (title,
  problem description, file location, consequences, reproduction steps, possible solutions),
  two publication modes (tmp/issues/ for GitHub-managed tracking, docs/issues/ for
  source-versioned manual tracking), file naming conventions, issue type prefixes, and
  ready-to-use templates for bug reports, feature requests, documentation gaps, and
  refactoring proposals.
category: kozen-wiki-writer
tags:
  - issues
  - bug-report
  - feature-request
  - publication-mode
  - project-management
  - tmp/issues
  - docs/issues
---

# Kozen Wiki — GitHub Issues Management

---

## Scope and rationale

GitHub Issues and wiki pages share the same decisions: where does the file live, who controls
versioning, and how does it fit into the project's documentation strategy. Keeping this
reference in the wiki-writer skill avoids loading a second skill for project-management tasks.

---

## Publication modes for issues

### Mode A — GitHub-managed (`tmp/issues/`)

The developer drafts issue files locally in `tmp/issues/` and publishes them through the
GitHub web UI or GitHub CLI. Once published, the canonical record lives in GitHub Issues —
the local file is a staging artifact and is not committed to the main repository.

**When to choose Mode A:**
- The team tracks work in GitHub Issues and wants search, assignment, labels, and milestones.
- Issues should be visible to external contributors without access to the source repository.
- You want to use `gh issue create` to publish directly from the CLI.

**Directory structure:**
```
tmp/
└── issues/
    ├── bug_auth-token-expires-silently.md
    ├── feat_add-x509-cert-rotation.md
    └── docs_missing-mcp-tool-schema.md
```

**Files in `tmp/issues/` are not committed.** Add this line to `.gitignore`:
```
tmp/
```

**Publishing with GitHub CLI:**
```bash
# Create from file body
gh issue create \
  --title "Bug: auth token expires silently without error" \
  --body-file tmp/issues/bug_auth-token-expires-silently.md \
  --label "bug" \
  --label "auth"
```

---

### Mode B — Source-versioned (`docs/issues/` or `doc/issues/`)

The developer commits issue files alongside source code in the project's documentation
directory. This is the canonical record — no GitHub Issues needed.

**When to choose Mode B:**
- The team prefers all project knowledge in the repository (monorepo, offline, or private).
- Issues double as architectural decision records (ADRs) or post-mortems.
- The project does not use GitHub Issues at all.

**Directory detection rule:** use `docs/issues/` if the project has a `docs/` directory;
use `doc/issues/` if the project uses `doc/` instead. Ask the user if neither exists.

**Directory structure:**
```
docs/
└── issues/
    ├── bug_auth-token-expires-silently.md
    ├── feat_add-x509-cert-rotation.md
    └── docs_missing-mcp-tool-schema.md
```

**Files in `docs/issues/` are committed** and appear in the repository alongside source code.

---

## File naming convention

```
<type>_<short-kebab-description>.md
```

| Type prefix | Meaning |
|---|---|
| `bug` | Something is broken or produces incorrect results |
| `feat` | A new feature or capability is requested |
| `docs` | Documentation is missing, incorrect, or misleading |
| `chore` | Maintenance: dependency updates, CI fixes, build scripts |
| `refactor` | Internal restructuring with no behaviour change |
| `perf` | A performance improvement without API changes |
| `test` | Missing or failing tests |
| `security` | A vulnerability or security concern |

Rules:
- All lowercase, no spaces.
- Prefix with `type_`.
- Description: 3 to 6 words, kebab-cased.
- Maximum 60 characters total (fits comfortably as a GitHub issue URL slug).

Examples:
```
bug_change-stream-closes-on-empty-db.md
feat_add-aws-sts-assume-role-support.md
docs_missing-x509-configuration-example.md
security_master-key-logged-in-debug-mode.md
```

---

## Standard issue structure

Every issue file, regardless of type, follows this six-section layout. Sections are
separated by horizontal rules. All sections are required; write `N/A` only if a field
genuinely cannot be filled (for example, reproduction steps for a documentation gap).

```markdown
# <Type>: <Short imperative title>

## 📋 Problem Description

<One to three paragraphs. What is wrong or missing? Be specific and factual.>

---

## 📍 Location

<File path(s), line numbers, function names, or configuration keys where the problem
originates. For documentation issues, the wiki page URL or README section.>

---

## ⚠️ Consequences

<What breaks, degrades, or risks for users if this is not addressed? Include severity:
Critical / High / Medium / Low.>

---

## 🔁 Reproduction Steps

<Numbered list. Exact commands, environment variable values, and input data needed to
reproduce the problem. For non-reproducible issues (missing docs, style), write N/A.>

---

## 💡 Possible Solutions

<Numbered list of candidate solutions with trade-offs. Recommend one if you can.>

---

## 📚 References

<Links to relevant source files, wiki pages, npm docs, MongoDB docs, or related issues.>
```

---

## Issue templates by type

### Bug report

```markdown
# Bug: <what breaks in a few words>

## 📋 Problem Description

`@kozen/<module>` v<version> <description of the incorrect behaviour>. Expected: <what
should happen>. Actual: <what does happen>.

---

## 📍 Location

- File: `src/<path>/<file>.ts`, line <n> — `<ClassName>.<methodName>()`
- Triggered by: `--action=<alias>:<method>` with `<flag>=<value>`

---

## ⚠️ Consequences

**Severity: High**

<Explain the user-facing impact. Data loss? Silent failure? Wrong output?>

---

## 🔁 Reproduction Steps

1. Install `@kozen/<module>@<version>`.
2. Set the following environment variables:
   ```
   KOZEN_<VAR>=<value>
   ```
3. Run:
   ```bash
   npx kozen --moduleLoad=@kozen/<module> --action=<alias>:<method> --<flag>=<value>
   ```
4. Observe: <what you see>.

---

## 💡 Possible Solutions

1. **<Option A>** — <brief description, trade-offs>.
2. **<Option B>** — <brief description, trade-offs>.

Recommended: Option A, because <reason>.

---

## 📚 References

- [Source file](https://github.com/<org>/<repo>/blob/main/src/<path>/<file>.ts)
- [Related wiki page](https://github.com/<org>/<repo>/wiki/<PageName>)
```

---

### Feature request

```markdown
# Feat: <what capability to add in a few words>

## 📋 Problem Description

Currently, `@kozen/<module>` does not support <capability>. Developers who need <use
case> must <workaround>. This adds friction and error potential.

---

## 📍 Location

The change would primarily affect:
- `src/<path>/<file>.ts` — <what needs to change>
- `src/configs/ioc.json` — <new token, if any>
- `src/docs/<module>.txt` — new flag or option documentation

---

## ⚠️ Consequences

**Severity: Medium**

Without this feature: <impact on developers or operations>.

---

## 🔁 Reproduction Steps

N/A — this is a feature request, not a defect.

---

## 💡 Possible Solutions

1. **<Approach A>** — <brief description>. Pros: <…>. Cons: <…>.
2. **<Approach B>** — <brief description>. Pros: <…>. Cons: <…>.

---

## 📚 References

- [Relevant npm package](https://www.npmjs.com/package/@kozen/<module>)
- [Kozen Engine wiki](https://github.com/kozen-labs/engine/wiki)
```

---

### Documentation gap

```markdown
# Docs: <what is missing or incorrect in a few words>

## 📋 Problem Description

The <wiki page / README / CLI help file> does not document <what is missing>. A developer
following the documentation <what breaks or what they cannot do>.

---

## 📍 Location

- Wiki page: `<PageName>.md` — section: `<Section Title>`
- Or: `README.md`, line <n>
- Or: `src/docs/<module>.txt`, section `<heading>`

---

## ⚠️ Consequences

**Severity: Low / Medium**

Developers must read source code to discover <undocumented behaviour>. Risk of incorrect
usage, especially for <specific scenario>.

---

## 🔁 Reproduction Steps

N/A

---

## 💡 Possible Solutions

1. Add a section `<Section Title>` to `<PageName>.md` covering: <list of missing items>.
2. Add an example showing <specific scenario> to the CLI page.

---

## 📚 References

- [Wiki page](https://github.com/<org>/<repo>/wiki/<PageName>)
- [Source](https://github.com/<org>/<repo>/blob/main/src/<path>/<file>.ts)
```

---

### Security concern

```markdown
# Security: <what is exposed or at risk in a few words>

## 📋 Problem Description

`@kozen/<module>` <description of the security concern>. This could allow <threat
scenario>.

---

## 📍 Location

- File: `src/<path>/<file>.ts`, line <n>
- Exposed via: <CLI flag / env var / log output / HTTP endpoint>

---

## ⚠️ Consequences

**Severity: Critical / High**

<Describe who can be affected, under what conditions, and what data or access is at risk.>

---

## 🔁 Reproduction Steps

1. <Step 1>
2. <Step 2>
3. Observe that `<sensitive data>` appears in `<output / logs / network traffic>`.

---

## 💡 Possible Solutions

1. **<Mitigation A>** — <description, compatibility impact>.
2. **<Mitigation B>** — <description, compatibility impact>.

---

## 📚 References

- [OWASP: Sensitive Data Exposure](https://owasp.org/www-community/vulnerabilities/Sensitive_Data_Exposure)
- [Source file](https://github.com/<org>/<repo>/blob/main/src/<path>/<file>.ts)
```

---

## Title conventions

Issue titles follow the same pattern as commit messages and PR titles.

**Format:** `<Type>: <short imperative phrase>`

Rules:
- Begin with the type prefix, capitalised, followed by a colon and a space.
- Write the description in the imperative mood: "Add", "Fix", "Remove", "Document" — not "Added", "Fixes", "Removing".
- Under 72 characters so the full title is readable in GitHub's issue list.
- No full stop at the end.

| Good | Avoid |
|---|---|
| `Bug: change stream closes on empty collection` | `bug: the change stream is being closed when...` |
| `Feat: add STS AssumeRole support for AWS secrets` | `Feature Request: STS support` |
| `Docs: missing X.509 certificate configuration example` | `Documentation issue with X509` |
| `Security: master key written to debug log` | `key logging vulnerability (urgent)` |

---

## Ask the user before creating issue files

Before generating any issue file, confirm:

**1. Publication mode**
> A) GitHub Issues — stage in `tmp/issues/`, publish via GitHub web UI or `gh issue create`
> B) Manual docs — commit to `docs/issues/` or `doc/issues/` alongside source code
> C) Not needed for this project

**2. Issue type** — bug, feat, docs, chore, refactor, perf, test, or security?

**3. Module and version** — which `@kozen/<module>` and version does this affect?

---

## Checklist before marking an issue file complete

- [ ] Title follows `<Type>: <imperative phrase>` format, under 72 characters
- [ ] Problem Description is factual and specific — no vague language
- [ ] Location includes at least one file path or wiki page URL
- [ ] Consequences includes a Severity label
- [ ] Reproduction Steps are numbered and copy-pasteable (or explicitly N/A)
- [ ] Possible Solutions lists at least one option with trade-offs
- [ ] References section links to the relevant source file and wiki page
- [ ] File name follows `<type>_<short-kebab-description>.md` convention
- [ ] File is in the correct directory for the chosen publication mode
