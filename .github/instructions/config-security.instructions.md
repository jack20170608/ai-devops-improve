---
applyTo: "**/*.properties,**/*.yml,**/*.yaml,**/*.json,**/*.env,**/Dockerfile,**/*.tf"
---

# Configuration security instructions

Human-readable source: `10-Devops-SOP/03-secure-coding-standard.md`.

## Secrets

Never place a real secret in configuration, source, tests, logs, documentation, prompts, instructions, skills, issue text, or MCP configuration.

Secrets include passwords, API keys, tokens, private keys, connection strings, webhook secrets, signing keys, and real customer data.

Unsafe:

```properties
spring.datasource.username=payments_admin
spring.datasource.******=<plaintext-password>
provider.api-key=<plaintext-api-key>
```

Safe:

```yaml
database:
  secretRef: secret-manager://payments/prod/database
  role: payments-api
```

- Production values must come from an approved secret manager, KMS, or workload identity.
- Prefer OIDC and short-lived credentials.
- Example files may contain only clearly invalid placeholders or secret references.
- Do not encode a secret with Base64 or URL encoding and call it safe.
- If a secret is exposed, tell the user to revoke or rotate it immediately; deleting the text is insufficient.

## Environment separation

- Keep development, test, staging, and production identities separate.
- Do not commit production hostnames, tenant data, credentials, or customer identifiers unless explicitly approved as non-sensitive.
- Defaults must be safe: no debug mode, anonymous administration, trust-all TLS, or unrestricted network access.
- Required production settings must fail fast when absent; do not silently fall back to development values.
- Document each setting's purpose, format, default, sensitivity, and restart behavior.

## Network and TLS

- Use HTTPS/TLS for sensitive traffic.
- Never disable certificate or hostname verification.
- Restrict outbound destinations with allowlists and platform egress policy.
- Do not allow user-controlled URLs to reach loopback, link-local, private, metadata, multicast, or internal addresses without an approved design.
- Set connection, read, write, and total-operation timeouts.

## Parsing and resources

- Limit request, file, document, archive, decompression, and nesting sizes.
- Constrain deserialization types.
- Do not enable unsafe polymorphic deserialization globally.
- Run containers as a non-root user where supported.
- Use read-only filesystems and drop unused capabilities where practical.
- Do not bake secrets into container image layers or build arguments.

## Review

Before completion:

- Check the diff for secrets and production values.
- Confirm secret references exist without revealing values.
- Confirm defaults fail safely.
- Confirm environment-specific values are not mixed.
- Confirm TLS verification, timeout, resource, and egress controls remain enabled.
- Run configuration validation and secret scanning.

