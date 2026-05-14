---
name: replicated-registries
description: This skill should be used when the user asks to "add a registry", "list registries", "test a registry", "remove a registry", "DockerHub registry", "ECR registry", "GCR registry", "GHCR registry", "Quay registry", or mentions Replicated private registry management, registry credentials, or proxy registry.
version: 0.1.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,registry,docker,ecr,gcr,ghcr,quay,images
  globs: ""
  alwaysApply: "false"
---

# Replicated Registries

Manage private registry connections for the Replicated proxy registry.

## Overview

The Replicated **proxy registry** allows you to distribute private images to customer installations without exposing your registry credentials. Connect your registries to Replicated so images can be proxied securely.

## Commands

### List Registries

```bash
replicated registry ls
replicated registry ls --output json
```

### Add Registries

```bash
# DockerHub
replicated registry add dockerhub --username <user> --password <pass>

# Amazon ECR
replicated registry add ecr --aws-access-key-id <id> --aws-secret-access-key <key> --region <region>

# Google Container Registry (GCR)
replicated registry add gcr --json-key <path-to-service-account.json>

# Google Artifact Registry (GAR)
replicated registry add gar --json-key <path-to-service-account.json> --location <location>

# GitHub Container Registry (GHCR)
replicated registry add ghcr --username <user> --password <token>

# Quay.io
replicated registry add quay --username <user> --password <pass>

# Generic / Other
replicated registry add other --endpoint <url> --username <user> --password <pass>
```

**Common flags:**
- `--skip-validation` — Skip validation (not recommended)

### Test Registry

```bash
replicated registry test <registry-name>
```

### Remove Registry

```bash
replicated registry rm <registry-name>
```

## Best Practices

- Always validate registry connections after adding
- Use service account tokens rather than personal credentials
- Test registry access before creating releases that depend on it
- Remove unused registries to keep the list clean

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Authentication failed" | Verify credentials; check token expiration |
| "Repository not found" | Verify repository name and permissions |
| "Network timeout" | Check firewall rules; verify registry endpoint |

## Further Reading

- **Replicated Proxy Registry**: https://docs.replicated.com/vendor/private-images-about
- **Replicated CLI Registry**: https://docs.replicated.com/reference/replicated-cli-registry
