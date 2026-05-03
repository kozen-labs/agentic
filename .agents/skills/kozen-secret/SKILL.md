---
name: kozen-secret
description: >
  Reference skill for @kozen/secret — a Kozen module that adds unified secret management
  across AWS Secrets Manager and MongoDB Client-Side Field Level Encryption (CSFLE). Covers
  the public ISecretManager interface, SecretManager, SecretManagerAWS, and SecretManagerMDB
  service classes, CLI commands (secret:get, secret:set, secret:metadata), MCP tools
  (kozen_secret_select, kozen_secret_save), all environment variables, how to use the module
  as a library dependency, and how to embed it as a programmatic service in other projects.
created: 2026-05-03
updated: 2026-05-03
---

# @kozen/secret — Secret Management Module

`@kozen/secret` extends the Kozen framework with a unified API for retrieving and storing
secrets across multiple backends. The same interface works with AWS Secrets Manager and
MongoDB Client-Side Field Level Encryption (CSFLE), allowing backend switching via a single
`--driver` flag or environment variable.

---

## Routing table

| Signal | Reference |
|---|---|
| ISecretManager, SecretManager, SecretManagerAWS, SecretManagerMDB, resolve secret, save secret, secret:get, secret:set, secret:metadata, kozen_secret_select, kozen_secret_save, MCP secret tool, CLI secret command, secret driver, aws driver, mdb driver | `references/api.md` |
| KOZEN_SM_KEY, KOZEN_SM_VAL, KOZEN_SM_DRIVER, KOZEN_SM_ALT, AWS_ACCESS_KEY_ID, MDB_MASTER_KEY, CSFLE, environment variables secret, configure secret module | `references/configuration.md` |

---

## When to use this skill

- Storing or retrieving credentials, API keys, or other secrets via CLI
- Exposing secret operations to an LLM via MCP
- Importing `SecretManager`, `SecretManagerAWS`, or `SecretManagerMDB` directly in a project
- Understanding the `ISecretManager` interface to implement a custom backend
- Setting up AWS Secrets Manager or MongoDB CSFLE as a secret store

---

## Quick start

```bash
npm install @kozen/secret
```

```bash
# Get a secret (MDB CSFLE driver)
npx kozen --moduleLoad=@kozen/secret --action=secret:get --key=MY_API_KEY --driver=mdb

# Get a secret (AWS Secrets Manager driver)
npx kozen --moduleLoad=@kozen/secret --action=secret:get --key=MY_API_KEY --driver=aws

# Save a secret
npx kozen --moduleLoad=@kozen/secret --action=secret:set --key=MY_API_KEY --value=abc123

# Show metadata
npx kozen --moduleLoad=@kozen/secret --action=secret:metadata
```

---

## Do & Don't

**Do:**
- Always pass the `--driver=mdb|aws` flag or set `KOZEN_SM_DRIVER` to select the backend.
- When using MongoDB CSFLE, set `MDB_MASTER_KEY` to a base64-encoded 96-byte key.
- Import `ISecretManager` (not the concrete class) when depending on this module in other code.
- Use `uriEnv` rather than embedding the connection string when using this module programmatically.

**Don't:**
- Never commit `MDB_MASTER_KEY` or AWS credentials to source control.
- Never use MongoDB CSFLE without a valid data encryption key already created in the key vault.
