# 🤖 Kozen Agentic Components

Skills and agents for the Kozen Task Execution Framework ecosystem. Install them with the
Agent Package Manager (APM) or copy them manually to activate context-aware AI assistance
for every Kozen workflow: building modules, managing secrets, running triggers, and auditing
MongoDB permissions.

---

## 📦 Contents

### Skills

Skills are structured context files that an AI coding assistant loads to reason about a
specific domain. Each skill in this repository covers one package in the Kozen ecosystem.

| Directory | npm package | Purpose |
|---|---|---|
| `kozen-engine` | `@kozen/engine` | Core framework: architecture, IoC container, module development guide, CLI and MCP interfaces |
| `kozen-secret` | `@kozen/secret` | Secret management: AWS Secrets Manager and MongoDB Client-Side Field Level Encryption (CSFLE) |
| `kozen-trigger` | `@kozen/trigger` | Self-hosted MongoDB Change Stream triggers: delegate files, event handlers, configuration |
| `kozen-iam-rectification` | `@kozen/iam-rectification` | MongoDB Identity and Access Management (IAM) permission validation for SCRAM and X.509 auth |

### Agent

The `kozen` agent (`kozen.md`) routes questions across all four skills automatically. Load
it when you want AI assistance on any Kozen topic without selecting a skill manually. The
agent identifies the relevant domain from the question and activates the matching skill.

---

## ⚙️ Installation with APM

APM (Agent Package Manager) handles installation to Claude Code, Cursor, or any
AGENTS.md-compatible tool.

**Step 1 — add this repository as a source:**

Edit your project's `apm.config.json` and add the following entry to `sources`:

```json
{
  "name": "kozen",
  "type": "github",
  "url": "https://github.com/kozen-labs/agentic",
  "ref": "main",
  "namespace": "kozen",
  "skillsPath": ".agents/skills",
  "agentsPath": ".agents/agents",
  "enabled": true
}
```

**Step 2 — clone the repository into the local cache:**

```bash
apm refresh kozen
```

**Step 3 — install to your preferred provider:**

```bash
# Claude Code (global, available in every session)
apm install --type skill --provider claude --scope global

# Cursor / VS Code (.mdc rules, project-local)
apm install --type skill --provider vscode --scope local

# Standard AGENTS.md convention
apm install --type skill --provider standard --scope global
```

**Step 4 — install the agent:**

```bash
apm install --type agent --provider claude --scope global
```

After installation, verify everything is in place:

```bash
apm status --type skill
apm status --type agent
```

---

## 🗂️ Manual installation

If you are not using APM, copy the directories directly.

**Claude Code (global):**

```bash
cp -r .agents/skills/kozen-engine     ~/.claude/skills/
cp -r .agents/skills/kozen-secret     ~/.claude/skills/
cp -r .agents/skills/kozen-trigger    ~/.claude/skills/
cp -r .agents/skills/kozen-iam-rectification ~/.claude/skills/
cp .agents/agents/kozen.md            ~/.claude/agents/
```

**Project-local (AGENTS.md convention):**

```bash
cp -r .agents/skills/kozen-engine     .agents/skills/
cp -r .agents/skills/kozen-secret     .agents/skills/
cp -r .agents/skills/kozen-trigger    .agents/skills/
cp -r .agents/skills/kozen-iam-rectification .agents/skills/
cp .agents/agents/kozen.md            .agents/agents/
```

---

## 🏗️ Repository structure

```
.agents/
├── skills/
│   ├── kozen-engine/
│   │   ├── SKILL.md                   ← routing table and overview
│   │   └── references/
│   │       ├── concepts.md            ← architecture, IoC, flow IDs, module lifecycle
│   │       ├── module-development.md  ← full guide: scaffold → publish
│   │       ├── configuration.md       ← config.json, env vars, template schema
│   │       └── cli-mcp.md             ← CLI options, MCP server config, logger API
│   ├── kozen-secret/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── api.md                 ← ISecretManager, CLI commands, MCP tools
│   │       └── configuration.md       ← env vars, driver comparison, key generation
│   ├── kozen-trigger/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── delegates.md           ← delegate format (ESM/CJS), ITriggerTools
│   │       └── configuration.md       ← KOZEN_TRIGGER_*, deployment patterns
│   └── kozen-iam-rectification/
│       ├── SKILL.md
│       └── references/
│           └── api.md                 ← interfaces, CLI flags, MCP tools, CI/CD pattern
└── agents/
    └── kozen.md                       ← routing agent for the full ecosystem
```

---

## 🔄 Keeping skills up to date

Check for newer versions:

```bash
apm outdated --type skill
```

Update all installed Kozen skills:

```bash
apm refresh kozen
apm install --type skill --provider claude --scope global
```

---

## 🤝 Contributing

This repository follows the Kozen contribution guidelines. To propose changes or report
issues, open a GitHub Issue with a clear description of the problem or improvement.

To add a new skill for an additional Kozen module:

1. Create a directory under `.agents/skills/` named `kozen-<module>`.
2. Add a `SKILL.md` with valid YAML frontmatter (`name`, `description`, `created`, `updated`).
3. Add reference files under `references/` as needed.
4. Update the `kozen` agent (`kozen.md`) routing table to include the new skill.
5. Open a Pull Request with the additions.

---

## 📚 References

[1] Kozen Engine. "@kozen/engine — Task Execution Framework." npm Registry, 2026.
    https://www.npmjs.com/package/@kozen/engine

[2] Kozen Labs. "Kozen Engine Wiki." GitHub, 2026.
    https://github.com/kozen-labs/engine/wiki

[3] Kozen Labs. "@kozen/secret." GitHub, 2026.
    https://github.com/mongodb-industry-solutions/kozen-secret/wiki

[4] Kozen Labs. "@kozen/trigger." GitHub, 2026.
    https://github.com/mongodb-industry-solutions/kozen-trigger/wiki

[5] Kozen Labs. "@kozen/iam-rectification." GitHub, 2026.
    https://github.com/mongodb-industry-solutions/kozen-iam-rectification/wiki

[6] Membrides Espinosa, Antonio. "APM — Agent Package Manager." GitHub, 2026.
    https://github.com/kozen-labs/agentic
