---
name: replicated-profiles
description: This skill should be used when the user asks to "manage Replicated profiles", "add a Replicated profile", "switch Replicated profiles", "set default profile", "list profiles", "remove a profile", "edit a profile", "manage RBAC policies", "list policies", "create a policy", or mentions Replicated CLI authentication profiles, multiple accounts, switching credentials, or RBAC policies.
version: 0.2.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,profile,auth,credentials,rbac,policy
  globs: ""
  alwaysApply: "false"
---

# Replicated Profiles

Manage authentication profiles and RBAC policies for the Replicated CLI.

## Overview

**Profiles** let you store multiple sets of credentials and easily switch between them. This is useful when working with different Replicated accounts (production, development, etc.) or different API endpoints.

Credentials are stored in `~/.replicated/config.yaml` with file permissions `600` (owner read/write only).

## Authentication Priority

When a command runs, credentials are resolved in this order:

1. `REPLICATED_API_TOKEN` environment variable (highest priority)
2. `--profile` flag (per-command override)
3. Default profile from `~/.replicated/config.yaml`
4. Legacy single token (backward compatibility)

## Commands

### Add a Profile

```bash
# Add a production profile (will prompt for token)
replicated profile add prod

# Add with token flag
replicated profile add prod --token=your-prod-token

# Add a development profile with custom API origin
replicated profile add dev \
  --token=your-dev-token \
  --api-origin=https://vendor-api-dev.com

# Add with custom registry origin
replicated profile add dev \
  --token=your-dev-token \
  --api-origin=https://vendor-api-dev.com \
  --registry-origin=vendor-registry-dev.com

# Add with Okteto namespace (auto-generates service URLs)
replicated profile add dev --namespace=noahecampbell
```

**Flags:**
- `--token` — API token (optional, will prompt if not provided)
- `--api-origin` — API origin URL (e.g., `https://api.replicated.com/vendor`)
- `--registry-origin` — Registry origin (e.g., `registry.replicated.com`)
- `--namespace` — Okteto namespace for dev environments (mutually exclusive with `--api-origin` and `--registry-origin`)

**Note:** If a profile with the same name already exists, `add` will update it.

### List Profiles

```bash
replicated profile ls
```

The default profile is indicated with an asterisk (`*`).

### Set Default Profile

```bash
# Use production as the default
replicated profile set-default prod

# Or use the alias
replicated profile use prod
```

The default profile is used when no `--profile` flag is specified and no `REPLICATED_API_TOKEN` environment variable is set.

### Edit a Profile

```bash
# Update the token for a profile
replicated profile edit dev --token=new-dev-token

# Update the API origin
replicated profile edit dev --api-origin=https://vendor-api-dev.com

# Update multiple fields at once
replicated profile edit dev \
  --token=new-token \
  --api-origin=https://vendor-api-dev.com \
  --registry-origin=vendor-registry-dev.com
```

Only the flags you provide will be updated; other fields remain unchanged.

### Remove a Profile

```bash
replicated profile rm dev
```

If the removed profile was the default, the default will be automatically set to another available profile (if any exist).

## Using Profiles in Commands

Any Replicated CLI command accepts the `--profile` flag to override the default for that single invocation:

```bash
# Use the prod profile for this command only
replicated release ls --profile prod

# Use the dev profile for a VM operation
replicated vm ls --profile dev

# Create a release using the staging profile
replicated release create --profile staging --yaml-dir ./manifests --version 1.2.3
```

This is especially useful in scripts and CI/CD where you need to target different environments without changing the default.

## Best Practices

- **Name profiles clearly** (`prod`, `dev`, `staging`) to avoid confusion
- **Use `--profile` in scripts** rather than changing the default
- **Set a safe default** — make `dev` or a non-production profile the default to prevent accidental prod operations
- **Rotate tokens periodically** via `replicated profile edit`
- **Avoid storing tokens in shell history** — use `replicated profile add <name>` (without `--token`) to be prompted securely

## RBAC Policies

Manage team access with RBAC policies.

### List Policies

```bash
replicated policy ls
```

### Get a Policy

```bash
replicated policy get <policy-id>
```

### Create a Policy

```bash
replicated policy create --name <name> --policy <policy-json>
```

### Update a Policy

```bash
replicated policy update <policy-id> --name <name> --policy <policy-json>
```

### Remove a Policy

```bash
replicated policy rm <policy-id>
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Profile not found" | Check with `replicated profile ls`; verify spelling |
| Wrong account targeted | Check active profile: `replicated profile ls`; use `--profile` explicitly |
| Token expired | Update with `replicated profile edit <name> --token=<new-token>` |
| Permission denied on `~/.replicated/config.yaml` | Ensure file permissions are `600` |

## Further Reading

- **Replicated CLI Profile**: https://docs.replicated.com/reference/replicated-cli-profile
- **Profile Use**: https://docs.replicated.com/reference/replicated-cli-profile-use
- **Profile Set Default**: https://docs.replicated.com/reference/replicated-cli-profile-set-default
