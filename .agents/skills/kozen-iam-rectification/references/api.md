---
name: "@kozen/iam-rectification — Public API"
description: >
  Full API reference for @kozen/iam-rectification: IRectificationOption and
  IRectificationOptionX509 input interfaces, IRectificationResponse output structure
  (extra/missing/present), IAMRectificationScram and IAMRectificationX509 concrete services,
  CLI verify command with all SCRAM and X.509 flags, MCP tools (kozen_iam_rectify_scram,
  kozen_iam_rectify_x509), IAMRectificationModule loading pattern, and programmatic
  TypeScript usage as a standalone library.
category: kozen-iam-rectification
tags:
  - kozen
  - IAM
  - MongoDB
  - SCRAM
  - X509
  - permissions
  - rectification
  - CLI
  - MCP
---

# @kozen/iam-rectification — Public API

## Public exports

```typescript
import {
  IAMRectificationModule,       // KzModule subclass — the module entry point
  IAMRectificationScram,        // Service: SCRAM-SHA-256 permission verification
  IAMRectificationX509,         // Service: X.509 certificate permission verification
  IRectificationOption,         // Input interface for SCRAM verification
  IRectificationOptionX509,     // Input interface for X.509 verification (extends IRectificationOption)
  IRectificationResponse,       // Output interface: extra/missing/present permissions
} from '@kozen/iam-rectification';
```

---

## IRectificationOption — SCRAM input interface

```typescript
export interface IRectificationOption {
  /** Full MongoDB connection string. Takes priority over uri built from parts. */
  uri?: string;

  /** Name of an environment variable that contains the connection URI. */
  uriEnv?: string;

  /** Hostname or cluster address (used to build URI when neither uri nor uriEnv is provided). */
  host?: string;

  /** Application name appended to connection string as appName. */
  app?: string;

  /** SCRAM username (used when building URI from parts). */
  username?: string;

  /** SCRAM password (used when building URI from parts). */
  password?: string;

  /** Authentication method label (e.g., 'SCRAM-SHA-256'). */
  method?: string;

  /** Protocol: 'mongodb' or 'mongodb+srv'. Defaults to 'mongodb+srv' when isCluster is true. */
  protocol?: string;

  /** Whether the connection target is a cluster (affects default protocol). */
  isCluster?: boolean;

  /**
   * Required: the set of permissions the application needs.
   * The service verifies which of these the user has and which are missing.
   * Default when omitted: ['search','read','find','insert','update','remove','collMod'].
   */
  permissions: string[];
}
```

### URI resolution rules

1. If `uri` is provided, it is used directly.
2. If `uriEnv` is provided, the value of `process.env[uriEnv]` is used.
3. If neither is provided, a URI is constructed from `protocol`, `username`, `password`,
   `host`, and `app`:
   - SCRAM: `protocol://[username:password@]host/?retryWrites=true&w=majority&appName=app`
4. If no URI can be resolved, an error is thrown.

---

## IRectificationOptionX509 — X.509 input interface

```typescript
export interface IRectificationOptionX509 extends IRectificationOption {
  /** PEM-encoded private key (inline string). */
  key?: string;

  /** PEM-encoded client certificate (inline string). */
  cert?: string;

  /** PEM-encoded CA certificate (inline string). */
  ca?: string;

  /** Path to a file containing the client certificate (alternative to inline cert). */
  certPath?: string;

  /** Path to a file containing the CA certificate (alternative to inline ca). */
  caPath?: string;
}
```

When both `cert` and `key` are provided (inline PEM), the service creates a TLS secure
context and uses it for the MongoDB connection. The `uri` for X.509 connections should not
include username/password — the certificate is the authentication credential.

---

## IRectificationResponse — output structure

```typescript
export interface IRectificationResponse {
  permissions: {
    /** Permissions the user has that are NOT in the required list. Remove to reduce attack surface. */
    extra: string[];

    /** Permissions in the required list that the user does NOT have. Grant to fix failures. */
    missing: string[];

    /** Permissions in the required list that the user DOES have. Correctly granted. */
    present: string[];
  };

  /** The resolved username of the verified user. */
  username?: string;

  /** All roles currently assigned to the user. */
  roles?: string[];
}
```

Example output:

```json
{
  "permissions": {
    "extra":   ["collMod", "dropCollection"],
    "missing": ["insert"],
    "present": ["read", "find"]
  },
  "username": "appUser@admin",
  "roles": ["readWrite", "clusterMonitor"]
}
```

---

## CLI command

### trigger: iam-rectification:verify

Both SCRAM and X.509 use the same `verify` action; the `--method` flag selects the path.

```bash
# SCRAM using a connection string from an environment variable
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=SCRAM \
  --uriEnv=MONGODB_URI \
  --permissions=read,find,insert

# SCRAM using a direct URI
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=SCRAM \
  --uri="mongodb+srv://user:pass@cluster.mongodb.net/db" \
  --permissions=read,find

# X.509 with inline cert paths
kozen --moduleLoad=@kozen/iam-rectification \
  --action=iam-rectification:verify \
  --method=X509 \
  --uri="mongodb+srv://cluster.mongodb.net/db" \
  --cert=/path/to/client.pem \
  --key=/path/to/client.key \
  --ca=/path/to/ca.pem \
  --permissions=read,find,insert

# Show help
kozen --moduleLoad=@kozen/iam-rectification --action=iam-rectification:help
```

### Module loading shorthand

```bash
# Using environment variables (recommended for CI)
export KOZEN_MODULE_LOAD=@kozen/iam-rectification
export KOZEN_STACK=prod
kozen --action=iam-rectification:verify --uriEnv=MONGO_PROD_URI --permissions=read,insert
```

### All CLI flags

**Core options:**

| Flag | Env var | Description |
|---|---|---|
| `--stack` | `KOZEN_STACK` | Environment id: `dev`, `test`, `prod` |
| `--project` | `KOZEN_PROJECT` | Project ID for log organization |
| `--config` | `KOZEN_CONFIG` | Path to config file |

**SCRAM / X.509 connection options:**

| Flag | Env var | Description |
|---|---|---|
| `--uri` | (none) | Full MongoDB connection string |
| `--uriEnv` | `KOZEN_IAM_URI_ENV` | Env var name containing the URI |
| `--method` | `KOZEN_IAM_METHOD` | `SCRAM` or `X509` |
| `--host` | (none) | Hostname / cluster for URI construction |
| `--app` | (none) | Application name |
| `--username` | (none) | SCRAM username (when building URI) |
| `--password` | (none) | SCRAM password (when building URI) |
| `--protocol` | (none) | `mongodb` or `mongodb+srv` |
| `--isCluster` | (none) | `true` / `false` |
| `--permissions` | (none) | Comma-separated required permissions |

**X.509-specific options:**

| Flag | Description |
|---|---|
| `--cert` | Path to X.509 client certificate PEM file |
| `--key` | Path to X.509 private key PEM file |
| `--ca` | Path to CA certificate PEM file |
| `--certPath` | Directory containing certificate files |
| `--caPath` | Directory containing CA certificate files |

---

## MCP tools

### kozen_iam_rectify_scram

```json
{
  "name": "kozen_iam_rectify_scram",
  "description": "Verify a MongoDB user's permissions using SCRAM-SHA-256 authentication.",
  "inputSchema": {
    "uri":         "string (optional) — full MongoDB connection string",
    "uriEnv":      "string (optional) — env var name containing the URI",
    "username":    "string (optional) — SCRAM username",
    "password":    "string (optional) — SCRAM password",
    "host":        "string (optional) — hostname or cluster",
    "permissions": "string[] — required permissions to verify"
  }
}
```

### kozen_iam_rectify_x509

```json
{
  "name": "kozen_iam_rectify_x509",
  "description": "Verify a MongoDB user's permissions using X.509 certificate authentication.",
  "inputSchema": {
    "uri":         "string (optional) — full MongoDB connection string",
    "uriEnv":      "string (optional) — env var name containing the URI",
    "cert":        "string (optional) — PEM certificate (inline or path)",
    "key":         "string (optional) — PEM private key (inline or path)",
    "ca":          "string (optional) — PEM CA cert (inline or path)",
    "certPath":    "string (optional) — directory containing cert files",
    "permissions": "string[] — required permissions to verify"
  }
}
```

---

## Programmatic TypeScript usage

### Using the Kozen IoC pattern

```typescript
import { IAMRectificationModule } from '@kozen/iam-rectification';

const mod = new IAMRectificationModule();
const deps = await mod.register({ type: 'cli' });

const assistant = /* your Kozen IIoC instance */;
await assistant.registerAll(deps);

// SCRAM verification
const scram = await assistant.resolve<import('@kozen/iam-rectification').IAMRectificationScram>(
  'iam-rectification:scram'
);
const result = await scram.rectify({
  uriEnv:      'MONGODB_URI',
  permissions: ['read', 'find', 'insert']
});
console.log(result);
// { permissions: { extra: [], missing: ['insert'], present: ['read', 'find'] }, ... }

// X.509 verification
const x509 = await assistant.resolve<import('@kozen/iam-rectification').IAMRectificationX509>(
  'iam-rectification:x509'
);
const report = await x509.rectify({
  uri:         'mongodb+srv://cluster.mongodb.net/',
  cert:        process.env.CLIENT_CERT_PEM!,
  key:         process.env.CLIENT_KEY_PEM!,
  ca:          process.env.CA_CERT_PEM!,
  permissions: ['read', 'find']
});
```

### Standalone usage without Kozen IoC

```typescript
import { IAMRectificationScram, IAMRectificationX509 } from '@kozen/iam-rectification';

// SCRAM
const scram = new IAMRectificationScram();
const result = await scram.rectify({
  uriEnv:      'MONGODB_URI',    // reads process.env.MONGODB_URI
  permissions: ['read', 'insert', 'update']
});

if (result.permissions.missing.length > 0) {
  console.error('Missing permissions:', result.permissions.missing);
  process.exit(1);
}

if (result.permissions.extra.length > 0) {
  console.warn('Over-privileged. Consider removing:', result.permissions.extra);
}

// X.509
const x509 = new IAMRectificationX509();
const report = await x509.rectify({
  uri:         process.env.MDB_URI!,
  certPath:    '/etc/ssl/mongodb-client',
  permissions: ['read', 'find']
});
```

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
        "KOZEN_MODULE_LOAD": "@kozen/iam-rectification",
        "KOZEN_LOG_LEVEL": "NONE",
        "MONGODB_URI": "mongodb+srv://audit-user:pass@cluster.mongodb.net/db"
      }
    }
  }
}
```

---

## CI/CD integration pattern

```yaml
# .github/workflows/iam-audit.yml
- name: Audit MongoDB permissions
  env:
    MONGODB_URI: ${{ secrets.MONGODB_PROD_URI }}
    KOZEN_MODULE_LOAD: "@kozen/iam-rectification"
    KOZEN_STACK: prod
  run: |
    npx kozen --action=iam-rectification:verify \
      --uriEnv=MONGODB_URI \
      --permissions=read,find,insert,update
    # Exit code is non-zero if missing permissions are detected
```

The CLI exits with a non-zero status code when `missing` permissions are found, making it
suitable as a blocking step in automated pipelines.
