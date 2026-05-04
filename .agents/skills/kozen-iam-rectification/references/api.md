---
name: "@kozen/iam-rectification — Public API"
description: >
  Full API reference for @kozen/iam-rectification: all exported classes and interfaces with
  complete constructor signatures and method documentation, IIAMRectification interface,
  IAMRectificationScram (SCRAM-SHA-256 with URI resolution rules), IAMRectificationX509
  (X.509 TLS with certificate options), IRectificationOption and IRectificationOptionX509
  full field reference, IRectificationResponse (extra/missing/present analysis), CLI verify
  command with all flags, MCP tools (kozen_iam_rectify_scram, kozen_iam_rectify_x509),
  programmatic TypeScript usage, standalone use, CI/CD integration, and MCP configuration.
category: kozen-iam-rectification
tags:
  - kozen
  - IAM
  - MongoDB
  - SCRAM
  - X509
  - permissions
  - rectification
  - IIAMRectification
  - IRectificationOption
  - IRectificationResponse
  - CLI
  - MCP
  - CI-CD
---

# @kozen/iam-rectification — Public API

## All public exports

```typescript
import {
  IAMRectificationModule,     // KzModule subclass — the Kozen module entry point
  IAMRectificationScram,      // Service: SCRAM-SHA-256 permission verification
  IAMRectificationX509,       // Service: X.509 certificate permission verification
  IIAMRectification,          // Interface: the contract both services implement
  IRectificationOption,       // Input interface: SCRAM connection + required permissions
  IRectificationOptionX509,   // Input interface: X.509 extension of IRectificationOption
  IRectificationResponse,     // Output interface: extra/missing/present permissions
  IRectificationScramArg,     // CLI arg interface (extends IArgs + IRectificationOption)
  IRectificationX509Arg,      // CLI arg interface (extends IArgs + IRectificationOptionX509)
} from '@kozen/iam-rectification';
```

---

## IAMRectificationModule

The Kozen module entry point.

**Constructor:**
```typescript
constructor(dependency?: any)
```
Reads `package.json` to populate `metadata` (name, version, summary, author, license, uri).
Sets `metadata.alias = 'iam-rectification'` for action routing.

**`register(config, opts?)`**
```typescript
register(config: IConfig | null, opts?: any): Promise<Record<string, IDependency> | null>
```
Returns the dependency map for the runtime type:
- `config.type === 'cli'` → `{ ...ioc, ...cli }` (both services + CLI controller)
- `config.type === 'mcp'` → `{ ...ioc, ...mcp }` (both services + MCP controller)
- default → `ioc` (IAMRectificationScram + IAMRectificationX509 only, no controllers)

IoC tokens registered:
- `iam-rectification:scram` → `IAMRectificationScram` (transient)
- `iam-rectification:x509`  → `IAMRectificationX509` (transient)
- `iam-rectification:controller:cli` → `RectificationCLIController` (CLI only)
- `iam-rectification:controller:mcp` → `RectificationMCPController` (MCP only)

---

## IIAMRectification — the primary interface

The contract both service implementations satisfy. Program against this interface to decouple
code from the specific authentication method:

```typescript
interface IIAMRectification {
  /**
   * Connect to MongoDB with the given options and compare the user's actual permissions
   * against the required permissions list.
   * Returns a structured report: extra (over-privileged), missing (under-privileged),
   * and present (correctly granted) permissions.
   */
  rectify(options: IRectificationOption): Promise<IRectificationResponse>;
}
```

---

## IRectificationOption — SCRAM input interface

All fields for SCRAM-SHA-256 connection and permission verification:

```typescript
interface IRectificationOption {
  /**
   * Full MongoDB connection string. Takes priority over uri built from parts.
   * Example: 'mongodb+srv://user:pass@cluster.mongodb.net/db'
   */
  uri?: string;

  /**
   * Name of an environment variable that contains the connection URI.
   * The service reads process.env[uriEnv] at runtime.
   * Example: 'MONGODB_URI' → reads process.env.MONGODB_URI
   * Preferred over uri for CI/CD and MCP scenarios to avoid credential exposure.
   */
  uriEnv?: string;

  /**
   * Hostname or cluster address. Used to construct a URI when neither uri nor uriEnv
   * is provided.
   * Example: 'cluster0.abcde.mongodb.net'
   */
  host?: string;

  /**
   * Application name. Appended as appName to the connection string.
   * Appears in MongoDB server logs for connection attribution.
   */
  app?: string;

  /**
   * SCRAM username. Used when building URI from host/username/password.
   */
  username?: string;

  /**
   * SCRAM password. Used when building URI from host/username/password.
   */
  password?: string;

  /**
   * Authentication method label. Informational; does not change driver behaviour.
   * Example: 'SCRAM-SHA-256'
   */
  method?: string;

  /**
   * Connection protocol. Default: 'mongodb+srv' when isCluster is true, 'mongodb' otherwise.
   */
  protocol?: string;

  /**
   * Whether the target is a cluster (Atlas or replica set).
   * Affects default protocol selection.
   */
  isCluster?: boolean;

  /**
   * Required: the set of permissions the application needs to function.
   * The service verifies which of these the user has (present), which are missing,
   * and which the user has that are not in this list (extra / over-privileged).
   * Default when omitted: ['search','read','find','insert','update','remove','collMod'].
   */
  permissions: string[];
}
```

### URI resolution priority

1. `uri` is set → use as-is.
2. `uriEnv` is set → read `process.env[uriEnv]`.
3. Neither → build from `protocol + '://' + [username:password@] + host + '/?...'`.
4. Nothing resolvable → throws an error.

---

## IRectificationOptionX509 — X.509 input interface

Extends `IRectificationOption` with certificate-based authentication fields:

```typescript
interface IRectificationOptionX509 extends IRectificationOption {
  /**
   * PEM-encoded private key (inline string).
   * Alternative to certPath — provide either the inline PEM or the file path.
   */
  key?: string;

  /**
   * PEM-encoded client certificate (inline string).
   * The certificate's subject (CN) becomes the MongoDB username for X.509 auth.
   */
  cert?: string;

  /**
   * PEM-encoded CA certificate (inline string).
   * Required if the server uses a self-signed or private CA.
   */
  ca?: string;

  /**
   * Path to a file or directory containing the client certificate PEM.
   * Loaded at runtime with fs.readFileSync().
   */
  certPath?: string;

  /**
   * Path to a file or directory containing the CA certificate PEM.
   * Loaded at runtime with fs.readFileSync().
   */
  caPath?: string;
}
```

For X.509 authentication:
- The MongoDB URI must NOT include username/password — the certificate is the credential.
- Use a URI of the form: `mongodb+srv://cluster.mongodb.net/db?authMechanism=MONGODB-X509`
- The service constructs a TLS secure context from `cert` + `key` + `ca` (or the file paths)
  and passes it to the MongoDB driver as `tls: true, tlsCertificateKeyFile / tlsCAFile`.

---

## IRectificationResponse — output structure

```typescript
interface IRectificationResponse {
  permissions: {
    /**
     * Permissions the user has that are NOT in the required list.
     * These are excess privileges — review and revoke to enforce least privilege.
     */
    extra: string[];

    /**
     * Permissions in the required list that the user does NOT have.
     * These must be granted before the application will work correctly.
     */
    missing: string[];

    /**
     * Permissions in the required list that the user DOES have.
     * These are correctly configured.
     */
    present: string[];
  };

  /** The username resolved during the connection (from SCRAM auth or X.509 CN). */
  username?: string;

  /**
   * All roles currently assigned to the user.
   * Format: 'roleName@database' (e.g., 'readWrite@mydb', 'clusterMonitor@admin').
   */
  roles?: string[];
}
```

**Interpreting the response:**
- `missing.length > 0` → application will fail due to insufficient permissions. **Grant** these.
- `extra.length > 0` → application is over-privileged. **Revoke** these for least-privilege compliance.
- `present` → correctly configured. No action needed.
- `missing.length === 0 && extra.length === 0` → perfectly scoped. Ideal state.

---

## IAMRectificationScram

SCRAM-SHA-256 implementation. Connects with username/password credentials.

**Constructor:**
```typescript
constructor(dependency?: {
  assistant: IIoC;
  logger: ILogger;
})
```

**Methods:**

### `rectify(options)`
```typescript
rectify(options: IRectificationOption): Promise<IRectificationResponse>
```

1. Resolves the MongoDB URI from `options` (see URI resolution rules above).
2. Connects via `MongoClient` with SCRAM credentials.
3. Calls `MongoRoleManager.verifyPermissions()` (from `@mongodb-solution-assurance/iam-util`).
4. Returns the structured `IRectificationResponse`.

**Standalone use:**
```typescript
import { IAMRectificationScram } from '@kozen/iam-rectification';

const scram = new IAMRectificationScram();

const result = await scram.rectify({
  uriEnv:      'MONGODB_URI',               // reads process.env.MONGODB_URI
  permissions: ['read', 'find', 'insert', 'update']
});

console.log('Missing:', result.permissions.missing);
console.log('Extra:',   result.permissions.extra);
console.log('Present:', result.permissions.present);
console.log('User:',    result.username);
console.log('Roles:',   result.roles);
```

---

## IAMRectificationX509

X.509 certificate implementation. Connects using TLS client certificate authentication.

**Constructor:**
```typescript
constructor(dependency?: {
  assistant: IIoC;
  logger: ILogger;
})
```

**Methods:**

### `rectify(options)`
```typescript
rectify(options: IRectificationOptionX509): Promise<IRectificationResponse>
```

1. Reads certificate and key from inline PEM strings or from `certPath`/`caPath` files.
2. Constructs a TLS secure context.
3. Connects via `MongoClient` with `tls: true` and the certificate context.
4. Calls `MongoRoleManager.verifyPermissions()`.
5. Returns the structured `IRectificationResponse`.

**Standalone use:**
```typescript
import { IAMRectificationX509 } from '@kozen/iam-rectification';

const x509 = new IAMRectificationX509();

// Using file paths
const result = await x509.rectify({
  uri:         process.env.MDB_URI!,
  certPath:    '/etc/ssl/client.pem',
  caPath:      '/etc/ssl/ca.pem',
  permissions: ['read', 'find']
});

// Using inline PEM strings
const result2 = await x509.rectify({
  uri:  process.env.MDB_URI!,
  cert: process.env.CLIENT_CERT_PEM!,
  key:  process.env.CLIENT_KEY_PEM!,
  ca:   process.env.CA_CERT_PEM!,
  permissions: ['read', 'find']
});
```

---

## IRectificationScramArg and IRectificationX509Arg — CLI argument shapes

These interfaces extend both `IArgs` (Kozen CLI args) and the option interfaces.
They are used internally by the CLI controller's `fill()` method; rarely needed directly.

```typescript
interface IRectificationScramArg extends IArgs, IRectificationOption {}
interface IRectificationX509Arg   extends IArgs, IRectificationOptionX509 {}
```

---

## CLI command

Both SCRAM and X.509 use the same `verify` action; `--method` selects the path.
The `fill()` method appends the method to the action name internally:
`--action=iam-rectification:verify --method=SCRAM` → dispatches to `controller.verifySCRAM()`.

### SCRAM examples

```bash
# Verify using URI from environment variable (recommended)
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=SCRAM \
  --uriEnv=MONGODB_URI \
  --permissions=read,find,insert

# Verify using inline URI
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=SCRAM \
  --uri="mongodb+srv://appUser:pass@cluster.mongodb.net/mydb" \
  --permissions=read,find,insert,update

# Build URI from parts
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=SCRAM \
  --host=cluster0.abcde.mongodb.net \
  --username=appUser \
  --password=mypass \
  --isCluster=true \
  --permissions=read,find
```

### X.509 examples

```bash
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=X509 \
  --uri="mongodb+srv://cluster.mongodb.net/mydb?authMechanism=MONGODB-X509" \
  --cert=/etc/ssl/client.pem \
  --key=/etc/ssl/client.key \
  --ca=/etc/ssl/ca.pem \
  --permissions=read,find,insert
```

### Help

```bash
kozen --moduleLoad=@kozen/iam-rectification --action=iam-rectification:help
```

### All CLI flags

**Core:**

| Flag | Env var | Description |
|---|---|---|
| `--method=SCRAM\|X509` | `KOZEN_IAM_METHOD` | Authentication method (required) |
| `--permissions=<list>` | — | Comma-separated required permissions |
| `--stack=<env>` | `KOZEN_STACK` | Environment: dev/test/prod |
| `--project=<id>` | `KOZEN_PROJECT` | Project ID for log correlation |
| `--config=<path>` | `KOZEN_CONFIG` | Path to cfg/config.json |

**Connection:**

| Flag | Env var | Description |
|---|---|---|
| `--uri=<uri>` | — | Full MongoDB connection string |
| `--uriEnv=<name>` | `KOZEN_IAM_URI_ENV` | Env var name containing the URI |
| `--host=<host>` | — | Hostname for URI construction |
| `--app=<name>` | — | Application name (appName in connection string) |
| `--username=<user>` | — | SCRAM username |
| `--password=<pass>` | — | SCRAM password |
| `--protocol=<proto>` | — | `mongodb` or `mongodb+srv` |
| `--isCluster=<bool>` | — | `true` if connecting to a cluster |

**X.509-specific:**

| Flag | Description |
|---|---|
| `--cert=<path>` | Path to client certificate PEM file |
| `--key=<path>` | Path to client private key PEM file |
| `--ca=<path>` | Path to CA certificate PEM file |
| `--certPath=<dir>` | Directory containing certificate files |
| `--caPath=<dir>` | Directory containing CA certificate files |

---

## MCP tools

### `kozen_iam_rectify_scram`

```json
{
  "name": "kozen_iam_rectify_scram",
  "description": "Verify a MongoDB user's permissions using SCRAM-SHA-256 authentication. Returns which required permissions are missing, extra, and present.",
  "inputSchema": {
    "uri":         "string (optional) — full MongoDB connection string",
    "uriEnv":      "string (optional) — env var name containing the URI",
    "username":    "string (optional) — SCRAM username",
    "password":    "string (optional) — SCRAM password",
    "host":        "string (optional) — hostname or cluster address",
    "app":         "string (optional) — application name",
    "permissions": "string[] — required permissions to verify (e.g. ['read','find','insert'])"
  }
}
```

### `kozen_iam_rectify_x509`

```json
{
  "name": "kozen_iam_rectify_x509",
  "description": "Verify a MongoDB user's permissions using X.509 certificate authentication. Returns which required permissions are missing, extra, and present.",
  "inputSchema": {
    "uri":         "string (optional) — full MongoDB connection string",
    "uriEnv":      "string (optional) — env var name containing the URI",
    "cert":        "string (optional) — path to PEM client certificate file",
    "key":         "string (optional) — path to PEM private key file",
    "ca":          "string (optional) — path to PEM CA certificate file",
    "certPath":    "string (optional) — directory containing cert files",
    "caPath":      "string (optional) — directory containing CA cert files",
    "permissions": "string[] — required permissions to verify"
  }
}
```

---

## Programmatic use patterns

### Pattern 1: Standalone — audit before deployment

```typescript
import { IAMRectificationScram, IRectificationResponse } from '@kozen/iam-rectification';

async function auditPermissions(): Promise<void> {
  const service = new IAMRectificationScram();

  const result: IRectificationResponse = await service.rectify({
    uriEnv:      'MONGODB_URI',
    permissions: ['read', 'find', 'insert', 'update', 'remove']
  });

  if (result.permissions.missing.length > 0) {
    console.error('FAIL: Missing permissions:', result.permissions.missing);
    process.exit(1);
  }

  if (result.permissions.extra.length > 0) {
    console.warn('WARN: Over-privileged:', result.permissions.extra);
  }

  console.log('PASS: All required permissions present.');
  console.log('User:', result.username, '| Roles:', result.roles?.join(', '));
}

auditPermissions();
```

### Pattern 2: Kozen IoC container

```typescript
import { IoC } from '@kozen/engine';
import { IAMRectificationModule, IIAMRectification } from '@kozen/iam-rectification';

const container = new IoC();
const mod = new IAMRectificationModule();
const deps = await mod.register(null);  // SDK mode
await container.register(deps!);

// SCRAM
const scram = await container.resolve<IIAMRectification>('iam-rectification:scram');
const report = await scram.rectify({ uriEnv: 'PROD_URI', permissions: ['read', 'find'] });

// X.509
const x509 = await container.resolve<IIAMRectification>('iam-rectification:x509');
const report2 = await x509.rectify({
  uri:      process.env.MDB_URI!,
  certPath: '/certs/client.pem',
  caPath:   '/certs/ca.pem',
  permissions: ['read', 'find']
});
```

### Pattern 3: CI/CD pipeline gate

```yaml
# .github/workflows/permissions-audit.yml
name: MongoDB Permission Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '18' }
      - name: Audit MongoDB permissions
        env:
          MONGODB_URI:       ${{ secrets.MONGODB_URI }}
          KOZEN_MODULE_LOAD: "@kozen/iam-rectification"
          KOZEN_STACK:       prod
        run: |
          npx kozen --action=iam-rectification:verify \
            --method=SCRAM \
            --uriEnv=MONGODB_URI \
            --permissions=read,find,insert,update,remove
          # Non-zero exit code if missing permissions are found
```

The CLI exits with code 1 when `missing.length > 0`, making it a natural pipeline gate.

### Pattern 4: MCP — AI-assisted permission auditing

Configure the MCP server and let an AI assistant audit permissions on demand:

```json
{
  "mcpServers": {
    "kozen-iam": {
      "command": "npx",
      "args": ["kozen"],
      "env": {
        "KOZEN_APP_TYPE":    "mcp",
        "KOZEN_MODULE_LOAD": "@kozen/iam-rectification",
        "KOZEN_LOG_LEVEL":   "NONE",
        "MONGODB_URI":       "mongodb+srv://audit-user@cluster.mongodb.net/db"
      }
    }
  }
}
```

The AI can call `kozen_iam_rectify_scram` with a list of required permissions and interpret
the `extra`/`missing`/`present` output to recommend role changes.

---

## Default permissions list

When `permissions` is omitted from any call, the service uses this default set:

```
['search', 'read', 'find', 'insert', 'update', 'remove', 'collMod']
```

Always specify `permissions` explicitly in production. The default is a broad set that may
not match what your application actually needs.
