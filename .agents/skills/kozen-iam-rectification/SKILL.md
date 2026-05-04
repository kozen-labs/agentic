---
name: kozen-iam-rectification
description: >
  Reference skill for @kozen/iam-rectification — a Kozen module that validates and
  rectifies MongoDB user roles/permissions to ensure compliance with the principle of
  least privilege. Covers IRectificationOption and IRectificationOptionX509 interfaces,
  IRectificationResponse (extra/missing/present permissions), IAMRectificationScram and
  IAMRectificationX509 services, CLI commands (iam-rectification:verify), MCP tools
  (kozen_iam_rectify_scram, kozen_iam_rectify_x509), all environment variables and CLI
  flags, and programmatic use as a library in TypeScript projects.
created: 2026-05-03
updated: 2026-05-03
---

# @kozen/iam-rectification — MongoDB IAM Permission Validation

`@kozen/iam-rectification` verifies that a MongoDB user has exactly the permissions required
for an operation — no more, no less. It produces a structured report identifying missing
permissions (which would cause runtime failures), extra permissions (security over-privilege),
and present permissions (correctly granted). Supports both SCRAM-SHA-256 and X.509
certificate authentication.

---

## Routing table

| Signal | Reference |
|---|---|
| IAMRectificationModule, IIAMRectification, IAMRectificationScram, IAMRectificationX509, IRectificationOption, IRectificationOptionX509, IRectificationResponse, IRectificationScramArg, IRectificationX509Arg, all iam exports, rectify method, SCRAM rectification, X509 rectification, iam-rectification:verify, iam-rectification:help, kozen_iam_rectify_scram, kozen_iam_rectify_x509, MCP IAM tools, programmatic IAM, standalone IAM, library IAM, permissions extra missing present, URI resolution rules, X509 TLS context, audit permissions, CI/CD IAM gate, exit code missing permissions, KOZEN_IAM_METHOD, KOZEN_IAM_URI_ENV, --uriEnv, --permissions, --method, --cert, --key, --ca, --certPath, --caPath, --host, --app, --username, --password, --isCluster, --protocol, environment variables IAM, CLI flags IAM, default permissions list | `references/api.md` |

---

## When to use this skill

- Auditing MongoDB user permissions from the CLI
- Running IAM validation as a step in a CI/CD pipeline
- Exposing permission auditing to an LLM via MCP
- Importing `IAMRectificationScram` or `IAMRectificationX509` in a Node.js project
- Understanding the `IRectificationResponse` output format and how to act on it

---

## Quick start

```bash
npm install @kozen/iam-rectification
```

```bash
# Verify SCRAM user permissions
npx kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=SCRAM \
  --uriEnv=MONGODB_URI \
  --permissions=read,find,insert

# Verify X.509 user permissions
npx kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=X509 \
  --cert=/path/to/client.pem \
  --key=/path/to/client.key \
  --ca=/path/to/ca.pem \
  --uri=mongodb+srv://cluster.mongodb.net/ \
  --permissions=read,find
```

---

## Understanding the output

```json
{
  "permissions": {
    "extra":   ["collMod", "dropCollection"],
    "missing": ["insert"],
    "present": ["read", "find"]
  },
  "username": "appUser",
  "roles": ["readWrite", "clusterMonitor"]
}
```

| Field | Meaning | Action |
|---|---|---|
| `extra` | Privileges the user has but the application does not need | Remove to reduce attack surface |
| `missing` | Privileges the application needs but the user lacks | Grant to fix runtime failures |
| `present` | Privileges correctly granted | No action needed |

---

## Do & Don't

**Do:**
- Pass `permissions` as the exact set the application requires — the module compares against this list.
- Use `uriEnv` (env var name) rather than `uri` (inline string) to avoid embedding credentials in scripts.
- Run this check in CI on every role change to catch drift early.
- Use X.509 authentication for M2M services in production (no password in the connection string).

**Don't:**
- Never grant permissions based on the `extra` list report without reviewing them first.
- Never use this module as a runtime gate (e.g., "check permissions before every query") —
  it makes network calls and is designed for audit/reconciliation workflows, not hot paths.
