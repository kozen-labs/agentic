---
name: "@kozen/secret — Configuration"
description: >
  All environment variables for @kozen/secret: KOZEN_SM_KEY, KOZEN_SM_VAL, KOZEN_SM_DRIVER,
  KOZEN_SM_ALT, AWS credentials (AWS_REGION, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY),
  MongoDB CSFLE credentials (MDB_URI, MDB_MASTER_KEY), .env file setup, and security
  best practices for secret store configuration.
category: kozen-secret
tags:
  - kozen
  - secret-manager
  - environment-variables
  - AWS
  - MongoDB-CSFLE
  - MDB_MASTER_KEY
  - configuration
---

# @kozen/secret — Configuration

## Environment variables

### Module loading

```bash
KOZEN_MODULE_LOAD=@kozen/secret
```

Required when the secret module is not pre-configured in `cfg/config.json`.

### Secret operation variables

| Variable | CLI flag | Description |
|---|---|---|
| `KOZEN_SM_KEY` | `--key` | Secret key name to retrieve or create |
| `KOZEN_SM_VAL` | `--value` | Secret value for `secret:set` operations |
| `KOZEN_SM_DRIVER` | `--driver` | Backend: `mdb` (MongoDB CSFLE) or `aws` (AWS Secrets Manager) |
| `KOZEN_SM_ALT` | `--alt` | Alternate namespace or context prefix |

### MongoDB CSFLE variables

| Variable | Description |
|---|---|
| `MDB_URI` | Full MongoDB connection string including credentials and cluster address |
| `MDB_MASTER_KEY` | Base64-encoded 96-byte local master key for CSFLE encryption |

Generating a master key (run once, store securely):

```bash
node -e "console.log(require('crypto').randomBytes(96).toString('base64'))"
```

The master key encrypts the data encryption key (DEK) stored in the MongoDB key vault
collection. If the master key is lost or rotated incorrectly, all CSFLE-encrypted secrets
become permanently unreadable.

### AWS Secrets Manager variables

| Variable | Description |
|---|---|
| `AWS_REGION` | AWS region where secrets are stored (e.g., `us-east-1`) |
| `AWS_ACCESS_KEY_ID` | AWS IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret access key |
| `AWS_PROFILE` | AWS credentials profile (alternative to key/secret pair) |

---

## .env file example

```bash
# Module loading
KOZEN_MODULE_LOAD=@kozen/secret

# Default operation settings
KOZEN_SM_DRIVER=mdb
KOZEN_SM_KEY=my-api-key

# MongoDB CSFLE
MDB_URI=mongodb+srv://user:pass@cluster.mongodb.net/vault?retryWrites=true&w=majority
MDB_MASTER_KEY=<96-byte-base64-encoded-key>

# Logging
KOZEN_LOG_LEVEL=DEBUG
KOZEN_LOG_MDB_ENABLED=false
```

---

## Driver comparison

| Feature | `mdb` (MongoDB CSFLE) | `aws` (AWS Secrets Manager) |
|---|---|---|
| Backend | MongoDB collection + CSFLE | AWS managed service |
| Encryption | Client-side, local master key | AWS KMS managed keys |
| Key management | Self-managed (`MDB_MASTER_KEY`) | AWS KMS (auto-managed) |
| Dependencies | `mongodb`, `mongodb-client-encryption` | `@aws-sdk/client-secrets-manager` |
| Cost | MongoDB Atlas / self-hosted storage | AWS pricing per secret + API call |
| Rotation | Manual | Native AWS rotation policies |
| Best for | Self-hosted / air-gapped / MongoDB-first | Cloud-native AWS workloads |

---

## Security best practices

1. **Never commit master keys or AWS credentials.** Use environment variables loaded at
   process start, or reference a secret manager for bootstrap credentials.

2. **Use least-privilege IAM policies for AWS.** Grant only
   `secretsmanager:GetSecretValue` and `secretsmanager:PutSecretValue` for the specific
   secret ARNs your application needs.

3. **Rotate credentials regularly.** The secret module does not manage rotation; implement
   rotation outside of Kozen using AWS rotation lambdas or a scheduled job.

4. **Keep MDB_MASTER_KEY in a separate secure store.** Store the key in AWS Parameter Store
   (SecureString) or HashiCorp Vault, and inject it at process start only. Never embed it
   in the application codebase.

5. **Use KOZEN_LOG_LEVEL=NONE in MCP mode.** Any console output breaks the MCP JSON-RPC
   channel. Ensure the logger does not print secret values even at DEBUG level.
