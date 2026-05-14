---
name: replicated-customers
description: This skill should be used when the user asks to "list Replicated customers", "create a customer", "download a license", "inspect a customer", "view customer instances", or mentions Replicated customer management, license files, or entitlements.
version: 0.3.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,customers,licenses
  globs: ""
  alwaysApply: "false"
---

# Replicated Customers

Manage customers, licenses, and entitlements for Replicated applications.

## Overview

**Customers** represent organizations that install your application. Each customer is associated with a **channel** and receives a **license file** for installation.

## Authentication

Authenticate before using any CLI commands:

```bash
replicated login
```

API tokens: https://vendor.replicated.com/team/tokens

### Environment Variable Configuration (direnv)

**For this user's projects**, auth and app config are typically managed via `.envrc` at the project root, loaded by `direnv`:

```bash
export REPLICATED_API_TOKEN="your-api-token-here"
export REPLICATED_APP="app-slug"
```

- After creating or modifying `.envrc`, run `direnv allow` from the directory
- `direnv` loads/unloads vars automatically when entering/leaving directories
- This eliminates the need for `--app` flags in most commands

## Customer Commands

### List Customers

```bash
replicated customer ls --app <app-slug>
```

### Create a Customer

```bash
replicated customer create \
  --app <app-slug> \
  --name "Customer Name" \
  --channel <channel> \
  --email "admin@example.com" \
  --license-type trial \
  --expires-in 30d
```

**License types:** `dev`, `trial`, `paid`

### Inspect a Customer

```bash
replicated customer inspect <customer-id> --app <app-slug>
```

### Download License File

```bash
replicated customer download-license <customer-id> \
  --app <app-slug> \
  --output license.yaml
```

## Instance Management

### List Customer Instances

```bash
# All instances for an app
replicated instance ls --app <app-slug>

# Instances for a specific customer
replicated instance ls --app <app-slug> --customer <customer-id>
```

### Inspect an Instance

```bash
replicated instance inspect <instance-id> --app <app-slug>
```

## Adoption Metrics

Track release adoption via the vendor portal:
- Channel adoption rates
- Update latency (time from release to installation)
- Version distribution across customer base
- Update success/failure rates

## Best Practices

- Use `dev` licenses for internal testing
- Set appropriate expiration for `trial` licenses
- Associate customers with the correct channel
- Download and version-control license files for CI/CD

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Customer not found" | Verify customer ID; list with `replicated customer ls` |
| "License expired" | Regenerate or extend via vendor portal |

## Further Reading

- **Vendor Documentation**: https://docs.replicated.com/vendor
- **CLI Reference**: https://docs.replicated.com/reference/vendor-cli
