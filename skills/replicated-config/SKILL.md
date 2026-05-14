---
name: replicated-config
description: This skill should be used when the user asks to "initialize Replicated config", "setup .replicated config", "replicated config init", or mentions Replicated project configuration, auto-detecting Helm charts, or linting preferences.
version: 0.1.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,config,helm,setup
  globs: ""
  alwaysApply: "false"
---

# Replicated Config

Initialize and manage `.replicated` configuration files for your project.

## Overview

The `replicated config` command initializes a `.replicated` config file for your project. It auto-detects Helm charts and preflight specs, and guides you through common settings like app ID, chart paths, and linting preferences.

## Commands

### Initialize Config

```bash
# Interactive setup with auto-detection
replicated config init

# Non-interactive (use defaults and auto-detected values only)
replicated config init --non-interactive

# Skip auto-detection entirely
replicated config init --skip-detection
```

## What It Auto-Detects

- **Helm charts** in the project directory
- **Preflight specs** (Troubleshoot manifests)
- **KOTS manifests** (if present)

## Best Practices

- Run `replicated config init` when setting up a new project
- Use `--non-interactive` in CI/CD pipelines
- Review the generated `.replicated/config.yaml` before committing

## Further Reading

- **Replicated CLI Config**: https://docs.replicated.com/reference/replicated-cli-config
