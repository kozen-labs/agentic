---
name: "@kozen/secret — Public API"
description: >
  Full API reference for @kozen/secret: all exported classes and interfaces with complete
  constructor signatures and method documentation, ISecretManager interface, SecretManager
  router service (full method list), SecretManagerAWS (createClient, AWS SDK details),
  SecretManagerMDB (initClient, createDataKey, getKmsProviders, CSFLE internals), CLI
  commands with all options, MCP tools with input schemas, module loading, programmatic
  use as a library, and standalone use without the Kozen runtime.
category: kozen-secret
tags:
  - kozen
  - secret-manager
  - ISecretManager
  - SecretManager
  - SecretManagerAWS
  - SecretManagerMDB
  - AWS-Secrets-Manager
  - MongoDB-CSFLE
  - CSFLE
  - CLI
  - MCP
---

# @kozen/secret — Public API

## All public exports

```typescript
import {
  SecretModule,           // KzModule subclass — the Kozen module entry point
  ISecretArgs,            // CLI argument shape (extends IArgs)
  ISecretManagerOptions,  // Service configuration options
  ISecretManager,         // Interface: the contract all backends implement
  SecretManager,          // Router service: selects backend by driver flag
  SecretManagerAWS,       // Concrete: AWS Secrets Manager integration
  SecretManagerMDB,       // Concrete: MongoDB CSFLE integration
  SecretCLIController,    // CLI controller (rarely imported directly)
  SecretMCPController,    // MCP controller (rarely imported directly)
} from '@kozen/secret';
```

---

## SecretModule

The Kozen module entry point. Extend or instantiate to load `@kozen/secret` into any
Kozen IoC container.

**Constructor:**
```typescript
constructor(dependency?: any)
```
Reads `package.json` to populate `metadata` (name, version, summary, author, license, uri).
Sets `metadata.alias = 'secret'` for action routing (`--action=secret:get`).

**`register(config, opts?)`**
```typescript
register(config: IConfig | null, opts?: any): Promise<Record<string, IDependency> | null>
```
Returns the dependency map for the given runtime type:
- `config.type === 'cli'` → `{ ...ioc, ...cli }` (services + CLI controller)
- `config.type === 'mcp'` → `{ ...ioc, ...mcp }` (services + MCP controller)
- default → `ioc` only (services, no controllers — for library/SDK use)

---

## ISecretManagerOptions — configuration interface

All fields that configure a secret manager instance:

```typescript
interface ISecretManagerOptions {
  /** Backend driver: 'mdb' (MongoDB CSFLE) or 'aws' (AWS Secrets Manager). */
  driver?: 'mdb' | 'aws';

  /** Secret key name for single-key operations. */
  key?: string;

  /** Secret value for write operations. */
  value?: string;

  /** Alternate namespace or context string (module-specific usage). */
  alt?: string;

  /** Flow ID for distributed tracing in logs. */
  flow?: string;

  /** AWS-specific configuration. */
  cloud?: {
    region?:           string;   // AWS region (e.g., 'us-east-1')
    accessKeyId?:      string;   // AWS access key ID
    secretAccessKey?:  string;   // AWS secret access key
    profile?:          string;   // AWS CLI profile name (alternative to key/secret)
  };

  /** MongoDB CSFLE-specific configuration. */
  mdb?: {
    uri?:         string;   // MongoDB connection string
    masterKey?:   string;   // Base64-encoded 96-byte master encryption key
    database?:    string;   // Database for storing key vault documents
    collection?:  string;   // Key vault collection name
    keyAltName?:  string;   // Alternate name for the data encryption key
  };
}
```

## ISecretArgs — CLI argument interface

```typescript
interface ISecretArgs extends IArgs {
  key?:    string;   // --key flag (maps to KOZEN_SM_KEY env var)
  value?:  string;   // --value flag (maps to KOZEN_SM_VAL env var)
  driver?: string;   // --driver flag (maps to KOZEN_SM_DRIVER env var)
  alt?:    string;   // --alt flag (maps to KOZEN_SM_ALT env var)
}
```

---

## ISecretManager — the primary interface

The contract all backend implementations satisfy. Program against this interface to keep
code decoupled from the specific backend (AWS vs MongoDB):

```typescript
interface ISecretManager {
  /** Current configuration options. */
  options: ISecretManagerOptions;

  /**
   * Configure the manager before use.
   * Updates internal options; must be called before resolve()/save() if not using
   * environment variable defaults.
   */
  configure(options: ISecretManagerOptions): void;

  /**
   * Retrieve the secret value for the given key.
   * Returns null if the key does not exist.
   * Returns string | number | boolean depending on the stored type.
   */
  resolve(
    key: string,
    options?: ISecretManagerOptions
  ): Promise<string | null | undefined | number | boolean>;

  /**
   * Create or update the secret at key with the given value.
   * Returns true on success, false on failure.
   */
  save(
    key: string,
    value: string | Binary,
    options?: ISecretManagerOptions
  ): Promise<boolean>;
}
```

---

## SecretManager — the router service

`SecretManager` is registered in IoC as `secret:manager`. It reads `options.driver` and
delegates to `SecretManagerAWS` (`'aws'`) or `SecretManagerMDB` (`'mdb'`). Use this when
the backend should be selectable at runtime via a flag or environment variable.

**Constructor:**
```typescript
constructor(
  options?: ISecretManagerOptions,
  dep?: { assistant: IIoC; logger: ILogger }
)
```

**Methods:**

| Method | Signature | Description |
|---|---|---|
| `configure(options)` | `(ISecretManagerOptions) => void` | Updates internal options. Sets `driver`, `cloud`, `mdb` fields. |
| `resolve(key, options?)` | `(string, ISecretManagerOptions?) => Promise<string \| null \| ...>` | Delegates to the backend selected by `options.driver`. Falls back to `process.env.KOZEN_SM_DRIVER`. |
| `save(key, value, options?)` | `(string, string \| Binary, ISecretManagerOptions?) => Promise<boolean>` | Creates or updates the secret in the selected backend. |
| `getValue(key, options?)` | `(string, ISecretManagerOptions?) => Promise<string \| null \| ...>` | Internal: resolves the backend instance, then calls `resolve()` on it. |

**Programmatic use (with driver switching at runtime):**
```typescript
import { SecretManager } from '@kozen/secret';

const mgr = new SecretManager();

// Switch to AWS at runtime
mgr.configure({ driver: 'aws', cloud: { region: process.env.AWS_REGION! } });
const awsVal = await mgr.resolve('my/prod/api-key');

// Switch to MongoDB CSFLE
mgr.configure({ driver: 'mdb', mdb: { uri: process.env.MDB_URI!, masterKey: process.env.MDB_MASTER_KEY! } });
const mdbVal = await mgr.resolve('my-local-secret');
```

---

## SecretManagerAWS — AWS Secrets Manager

Stores and retrieves secrets from AWS Secrets Manager. Uses `@aws-sdk/client-secrets-manager`.

**Constructor:**
```typescript
constructor(
  options?: ISecretManagerOptions,
  dep?: { assistant: IIoC; logger: ILogger }
)
```

**Methods:**

| Method | Signature | Description |
|---|---|---|
| `resolve(key, options?)` | `(string, ISecretManagerOptions?) => Promise<string \| null \| ...>` | Retrieves the secret. AWS stores JSON objects; returns the full JSON string. For plain string secrets, returns the raw value. |
| `save(key, value, options?)` | `(string, string \| Binary, ISecretManagerOptions?) => Promise<boolean>` | Creates (`CreateSecretCommand`) or updates (`PutSecretValueCommand`) the secret. |
| `createClient(options)` | `(ISecretManagerOptions) => SecretsManagerClient` | Builds and returns an AWS SDK `SecretsManagerClient` with credentials from `options.cloud` or environment variables. |

**Credential resolution order:**
1. `options.cloud.accessKeyId` + `options.cloud.secretAccessKey`
2. `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` environment variables
3. `options.cloud.profile` → AWS credentials file
4. `AWS_PROFILE` environment variable
5. IAM instance profile (EC2/ECS/Lambda automatic)

**Standalone use:**
```typescript
import { SecretManagerAWS } from '@kozen/secret';

const aws = new SecretManagerAWS();
aws.configure({
  driver: 'aws',
  cloud: {
    region:          process.env.AWS_REGION!,
    accessKeyId:     process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
  }
});

// Retrieve
const apiKey = await aws.resolve('my-service/api-key');

// Create / update
await aws.save('my-service/api-key', 'new-secret-value');
```

---

## SecretManagerMDB — MongoDB CSFLE

Stores secrets as Client-Side Field Level Encrypted (CSFLE) documents in MongoDB. The
encryption key never leaves the application process — MongoDB only sees ciphertext.

**Constructor:**
```typescript
constructor(
  options?: ISecretManagerOptions,
  dep?: { assistant: IIoC; logger: ILogger }
)
```

**Properties:**

| Property | Type | Description |
|---|---|---|
| `client` | `MongoClient \| null` | Active MongoDB connection (managed internally) |
| `encryption` | `ClientEncryption \| null` | CSFLE encryption context (managed internally) |
| `kmsProviders` | `KMSProviders \| null` | KMS configuration object for Awilix |

**Methods:**

| Method | Signature | Description |
|---|---|---|
| `resolve(key, options?)` | `(string, ISecretManagerOptions?) => Promise<string \| null \| ...>` | Finds and decrypts the document stored at `key`. Returns the decrypted value string. |
| `save(key, value, options?)` | `(string, string \| Binary, ISecretManagerOptions?) => Promise<boolean>` | Inserts or replaces the encrypted document at `key`. |
| `initClient(options?)` | `(ISecretManagerOptions?) => Promise<MongoClient>` | Creates and connects a `MongoClient` with CSFLE auto-encryption options configured. |
| `createDataKey(options?)` | `(ISecretManagerOptions?) => Promise<any>` | Creates a new data encryption key in the key vault collection. Called automatically on first use if no key exists. |
| `getKeyAlt(options?)` | `(ISecretManagerOptions?) => string` | Returns the alternate key name used to look up the data encryption key. Defaults to `'master'`. |
| `getKeyVaultNamespace(options?)` | `(ISecretManagerOptions?) => string` | Returns the key vault namespace (`database.collection`). Default: `'kozen.keyVault'`. |
| `getKmsProviders(options?)` | `(ISecretManagerOptions?) => KMSProviders` | Builds the KMS providers config for CSFLE. Uses `options.mdb.masterKey` or `MDB_MASTER_KEY` env var. |
| `getLocalKey()` | `() => string` | Returns `process.env.MDB_MASTER_KEY`. Throws if not set. |
| `getOptions(options?)` | `(ISecretManagerOptions?) => ClientEncryptionOptions` | Builds the `ClientEncryptionOptions` object needed by the MongoDB driver. |
| `close()` | `() => Promise<void>` | Closes the MongoDB client. Call when done in long-running processes. |

**Master key generation (one-time setup):**
```bash
node -e "console.log(require('crypto').randomBytes(96).toString('base64'))"
```
Store the output as `MDB_MASTER_KEY` in a secure secrets store. **Never regenerate** — all
previously encrypted secrets become unreadable if the key changes.

**Standalone use:**
```typescript
import { SecretManagerMDB } from '@kozen/secret';

const mdb = new SecretManagerMDB();
mdb.configure({
  driver: 'mdb',
  mdb: {
    uri:       process.env.MDB_URI!,
    masterKey: process.env.MDB_MASTER_KEY!
  }
});

// Write an encrypted secret
await mdb.save('db-password', 'super-secret-value');

// Read it back (decrypted automatically)
const password = await mdb.resolve('db-password');

// Clean up connection
await mdb.close();
```

---

## CLI commands

### `secret:get` — retrieve a secret

```bash
kozen --moduleLoad=@kozen/secret --action=secret:get \
  --key=MY_SECRET_KEY \
  --driver=mdb       # or 'aws'

# Via environment variables
KOZEN_MODULE_LOAD=@kozen/secret KOZEN_SM_KEY=MY_KEY KOZEN_SM_DRIVER=mdb kozen --action=secret:get
```

### `secret:set` — store a secret

```bash
kozen --moduleLoad=@kozen/secret --action=secret:set \
  --key=MY_SECRET_KEY \
  --value="the-secret-value" \
  --driver=aws
```

### `secret:metadata` — inspect the secret store

```bash
kozen --moduleLoad=@kozen/secret --action=secret:metadata --driver=mdb
```

### `secret:help` — show help

```bash
kozen --moduleLoad=@kozen/secret --action=secret:help
```

### All CLI options

| Flag | Env var | Required | Description |
|---|---|---|---|
| `--key=<name>` | `KOZEN_SM_KEY` | For get/set | Secret key name |
| `--value=<val>` | `KOZEN_SM_VAL` | For set | Secret value to store |
| `--driver=mdb\|aws` | `KOZEN_SM_DRIVER` | Yes | Backend driver |
| `--alt=<ns>` | `KOZEN_SM_ALT` | No | Alternate namespace or context |
| `--stack=<env>` | `KOZEN_STACK` | No | Environment: dev/test/prod (default: dev) |
| `--project=<id>` | `KOZEN_PROJECT` | No | Project ID for log correlation |
| `--config=<path>` | `KOZEN_CONFIG` | No | Path to cfg/config.json |

---

## MCP tools

### `kozen_secret_select` — retrieve a secret

```json
{
  "name": "kozen_secret_select",
  "description": "Retrieve a secret value by key from the configured secret store.",
  "inputSchema": {
    "key":    "string — the secret key name (required)",
    "driver": "string — 'mdb' or 'aws' (optional, uses KOZEN_SM_DRIVER if omitted)"
  }
}
```

### `kozen_secret_save` — store a secret

```json
{
  "name": "kozen_secret_save",
  "description": "Create or update a secret in the configured secret store.",
  "inputSchema": {
    "key":    "string — the secret key name (required)",
    "value":  "string — the secret value (required)",
    "driver": "string — 'mdb' or 'aws' (optional)"
  }
}
```

---

## Programmatic use patterns

### Pattern 1: Load into a Kozen IoC container

```typescript
import { IoC } from '@kozen/engine';
import { SecretModule, ISecretManager } from '@kozen/secret';

const container = new IoC();

// Register the engine's built-in logger first (required dependency)
// (or use your host Kozen app's existing container)
const mod = new SecretModule();
const deps = await mod.register(null);  // null = SDK mode, no controllers
await container.register(deps!);

const mgr = await container.resolve<ISecretManager>('secret:manager');
mgr.configure({ driver: 'mdb', mdb: { uri: process.env.MDB_URI!, masterKey: process.env.MDB_MASTER_KEY! } });
const val = await mgr.resolve('MY_KEY');
```

### Pattern 2: Standalone (no Kozen runtime)

```typescript
import { SecretManagerMDB, SecretManagerAWS, ISecretManager } from '@kozen/secret';

function createSecretManager(driver: 'aws' | 'mdb'): ISecretManager {
  if (driver === 'aws') {
    const mgr = new SecretManagerAWS();
    mgr.configure({ driver: 'aws', cloud: { region: process.env.AWS_REGION! } });
    return mgr;
  }
  const mgr = new SecretManagerMDB();
  mgr.configure({ driver: 'mdb', mdb: { uri: process.env.MDB_URI!, masterKey: process.env.MDB_MASTER_KEY! } });
  return mgr;
}

const mgr = createSecretManager(process.env.SECRET_DRIVER as 'aws' | 'mdb' ?? 'mdb');
const dbPassword = await mgr.resolve('DB_PASSWORD');
```

### Pattern 3: Combined with @kozen/trigger (secrets in delegates)

```javascript
// mydelegate.mjs — delegate file for @kozen/trigger
// Start kozen with both modules: KOZEN_MODULE_LOAD=@kozen/secret,@kozen/trigger

export async function insert(change, tools) {
  // Resolve secret manager from the shared IoC container
  const secretMgr = await tools.assistant?.resolve('secret:manager');
  const apiKey = await secretMgr?.resolve('EXTERNAL_API_KEY');

  // Use the secret to call an external service
  await fetch('https://api.example.com/events', {
    method: 'POST',
    headers: { Authorization: `Bearer ${apiKey}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ event: change.fullDocument })
  });
}
```

### Pattern 4: MCP server for AI-driven secret management

```json
{
  "mcpServers": {
    "kozen-secret": {
      "command": "npx",
      "args": ["kozen"],
      "env": {
        "KOZEN_APP_TYPE":   "mcp",
        "KOZEN_MODULE_LOAD": "@kozen/secret",
        "KOZEN_LOG_LEVEL":  "NONE",
        "MDB_URI":          "mongodb+srv://...",
        "MDB_MASTER_KEY":   "<base64-key>",
        "KOZEN_SM_DRIVER":  "mdb"
      }
    }
  }
}
```

The AI can then call `kozen_secret_select` and `kozen_secret_save` tools to manage secrets
on behalf of the user.
