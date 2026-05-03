---
name: "@kozen/secret — Public API"
description: >
  Full API reference for @kozen/secret: ISecretManager interface, SecretManager router
  service, SecretManagerAWS and SecretManagerMDB concrete implementations, CLI commands
  with all options, MCP tools with input schemas, module loading and initialization,
  and programmatic usage as a library dependency.
category: kozen-secret
tags:
  - kozen
  - secret-manager
  - ISecretManager
  - AWS-Secrets-Manager
  - MongoDB-CSFLE
  - CLI
  - MCP
---

# @kozen/secret — Public API

## Public exports

```typescript
import {
  SecretModule,           // KzModule subclass — the module entry point
  ISecretArgs,            // CLI argument shape
  ISecretManagerOptions,  // Service configuration options
  ISecretManager,         // Interface: implemented by all backends
  SecretManager,          // Router service: selects backend by driver flag
  SecretManagerAWS,       // Concrete: AWS Secrets Manager
  SecretManagerMDB,       // Concrete: MongoDB CSFLE
  SecretCLIController,    // CLI controller (rarely imported directly)
  SecretMCPController,    // MCP controller (rarely imported directly)
} from '@kozen/secret';
```

---

## ISecretManager interface

The primary public contract. All backend implementations satisfy this interface.

```typescript
interface ISecretManager {
  /**
   * Configure the manager before use.
   * Must be called before resolve() or save() if not using default env var config.
   */
  configure(options: ISecretManagerOptions): void;

  /**
   * Retrieve the value associated with key.
   * Returns null if the key does not exist.
   */
  resolve(key: string): Promise<string | null>;

  /**
   * Create or update the secret at key with value.
   * Returns true on success.
   */
  save(key: string, value: string): Promise<boolean>;

  /**
   * Return backend-specific metadata about the configured store.
   */
  metadata(): Promise<any>;
}
```

---

## SecretManager — the router service

`SecretManager` is the default service registered in the IoC container as `secret:manager`.
It reads the `driver` configuration option and delegates to either `SecretManagerAWS`
(driver `aws`) or `SecretManagerMDB` (driver `mdb`). Use this class when you want the
driver to be selectable at runtime via a flag or environment variable.

```typescript
import { SecretManager, ISecretManagerOptions } from '@kozen/secret';

const manager = new SecretManager();
manager.configure({
  driver: 'mdb',  // or 'aws'
  key: 'MY_API_KEY',
  // ...provider-specific options
});

const value = await manager.resolve('MY_API_KEY');
const saved = await manager.save('MY_NEW_KEY', 'secret-value');
const meta  = await manager.metadata();
```

---

## SecretManagerAWS

Direct AWS Secrets Manager integration. Uses `@aws-sdk/client-secrets-manager`.

```typescript
import { SecretManagerAWS } from '@kozen/secret';

const manager = new SecretManagerAWS();
manager.configure({
  driver: 'aws',
  cloud: {
    region:          process.env.AWS_REGION!,
    accessKeyId:     process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
  }
});

const value = await manager.resolve('my-secret-name');
```

AWS stores secrets as JSON objects. The module retrieves the full JSON document and returns
the string representation. For single-value secrets (plain strings), the raw value is returned.

---

## SecretManagerMDB

MongoDB Client-Side Field Level Encryption (CSFLE) backend. Stores secrets as encrypted
documents in a MongoDB collection, encrypted with a local master key. The key material
never leaves the application process.

```typescript
import { SecretManagerMDB } from '@kozen/secret';

const manager = new SecretManagerMDB();
manager.configure({
  driver: 'mdb',
  uri:       process.env.MDB_URI!,
  masterKey: process.env.MDB_MASTER_KEY!  // base64-encoded 96-byte key
});

const value = await manager.resolve('my-secret-name');
```

The master key format: a base64-encoded 96-byte random string. Generate it once:

```bash
node -e "console.log(require('crypto').randomBytes(96).toString('base64'))"
```

Store the output in a secure location (e.g., AWS Parameter Store, 1Password) and set it as
`MDB_MASTER_KEY` in the process environment. Never regenerate it — all previously encrypted
secrets become unreadable if the key changes.

---

## CLI commands

### secret:get

Retrieve a secret value by key.

```bash
kozen --moduleLoad=@kozen/secret --action=secret:get \
  --key=MY_SECRET_KEY \
  --driver=mdb|aws

# Or via environment variables:
export KOZEN_MODULE_LOAD=@kozen/secret
export KOZEN_SM_KEY=MY_SECRET_KEY
export KOZEN_SM_DRIVER=mdb
kozen --action=secret:get
```

### secret:set

Create or update a secret.

```bash
kozen --moduleLoad=@kozen/secret --action=secret:set \
  --key=MY_SECRET_KEY \
  --value=my-secret-value \
  --driver=mdb|aws
```

### secret:metadata

Display metadata about the configured secret store.

```bash
kozen --moduleLoad=@kozen/secret --action=secret:metadata --driver=mdb
```

### CLI options for secret commands

| Option | Env var | Description |
|---|---|---|
| `--key=<name>` | `KOZEN_SM_KEY` | Secret key name |
| `--value=<val>` | `KOZEN_SM_VAL` | Secret value (for `secret:set`) |
| `--driver=mdb\|aws` | `KOZEN_SM_DRIVER` | Backend driver |
| `--alt=<ns>` | `KOZEN_SM_ALT` | Alternate namespace / context |

---

## MCP tools

### kozen_secret_select

Retrieve a secret by key.

```json
{
  "name": "kozen_secret_select",
  "description": "Retrieve a secret value by key from the configured secret store.",
  "inputSchema": {
    "key":    "string — the secret key name",
    "driver": "string (optional) — 'mdb' or 'aws'"
  }
}
```

### kozen_secret_save

Create or update a secret.

```json
{
  "name": "kozen_secret_save",
  "description": "Create or update a secret in the configured secret store.",
  "inputSchema": {
    "key":    "string — the secret key name",
    "value":  "string — the secret value",
    "driver": "string (optional) — 'mdb' or 'aws'"
  }
}
```

---

## Programmatic use as a library

### Load and use with the Kozen IoC container

```typescript
import { SecretModule } from '@kozen/secret';

const mod = new SecretModule();
const deps = await mod.register({ type: 'cli' });  // or null for SDK mode

// Wire into your Kozen app's IoC container
const assistant = /* your Kozen IIoC instance */;
await assistant.registerAll(deps);

// Resolve and use
const manager = await assistant.resolve<import('@kozen/secret').ISecretManager>('secret:manager');
const value = await manager.resolve('MY_API_KEY');
```

### Direct use without Kozen IoC (standalone)

```typescript
import { SecretManagerMDB, SecretManagerAWS } from '@kozen/secret';

// MongoDB CSFLE
const mdb = new SecretManagerMDB();
mdb.configure({ driver: 'mdb', uri: process.env.MDB_URI!, masterKey: process.env.MDB_MASTER_KEY! });
const val = await mdb.resolve('MY_KEY');

// AWS
const aws = new SecretManagerAWS();
aws.configure({ driver: 'aws', cloud: { region: 'us-east-1', ... } });
const val2 = await aws.resolve('my/aws/secret');
```

Use the concrete classes directly when you do not need runtime driver switching or when
integrating into a non-Kozen project that already has its own DI framework.

---

## MCP server configuration example

```json
{
  "servers": {
    "kozen": {
      "type": "stdio",
      "command": "npx",
      "args": ["kozen"],
      "env": {
        "KOZEN_APP_TYPE": "mcp",
        "KOZEN_MODULE_LOAD": "@kozen/secret",
        "KOZEN_LOG_LEVEL": "NONE",
        "MDB_URI": "mongodb+srv://<user>:<pass>@<cluster>/<db>",
        "MDB_MASTER_KEY": "<base64-96-byte-key>",
        "KOZEN_SM_DRIVER": "mdb"
      }
    }
  }
}
```
