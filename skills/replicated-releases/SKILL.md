---
name: replicated-releases
description: This skill should be used when the user asks to "create a Replicated release", "promote a release", "lint Replicated manifests", "package a Helm chart for Replicated", "manage release versions", or mentions Replicated release workflows, CI/CD pipelines, or semantic versioning for KOTS manifests.
version: 0.3.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,releases,helm,kots,ci-cd
  globs: ""
  alwaysApply: "false"
---

# Replicated Releases

Create, lint, promote, and manage Replicated application releases.

## Overview

A **release** is a versioned snapshot of application manifests (KOTS YAML, Helm charts, etc.). Releases are promoted to **channels** so customers can receive updates.

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

## Creating Releases

### Quick Workflow (Lint + Create + Promote)

```bash
replicated release create \
  --app <app-slug> \
  --yaml-dir <manifest-dir> \
  --lint \
  --promote <channel> \
  --version <version>
```

### Step-by-Step Workflow

```bash
# 1. Lint first
replicated release lint --app <app-slug> --yaml-dir <manifest-dir>

# 2. Create release (note the sequence number from output)
replicated release create \
  --app <app-slug> \
  --yaml-dir <manifest-dir> \
  --version <version>

# 3. Promote to channel
replicated release promote <sequence> <channel> \
  --app <app-slug> \
  --version <version>
```

### Linting

Always validate manifests before creating releases:

```bash
replicated release lint --app <app-slug> --yaml-dir <manifest-dir>
```

**Common lint issues:**
- Missing required fields in KOTS manifests
- Invalid Kubernetes resource specs
- Broken Helm template syntax
- Nonexistent status informer objects
- Schema validation failures

**Filter ignorable warnings:**
```bash
replicated release lint --app chef-360 --yaml-dir manifests | \
  grep -v 'nonexistent-status-informer-object'
```

## Version Labeling

### Semantic Versioning

```bash
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3
```

**When to increment:**
- `MAJOR` — Breaking changes or major feature releases
- `MINOR` — New features, backward-compatible
- `PATCH` — Bug fixes, security patches

### Pre-release Versions

```bash
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3-beta.1
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3-rc.2
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3-dev.20250119
```

### Build Metadata & Branch-Based Versioning

```bash
# Git commit SHA
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3+abc123

# Branch + SHA for CI/CD
VERSION="$(git branch --show-current)-$(git rev-parse --short HEAD)"
replicated release create --app myapp --yaml-dir ./manifests --version "$VERSION"
```

## Promoting Releases

### Promote by Sequence Number

```bash
# Create release (get sequence from output)
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3
# Output: SEQUENCE: 42

# Promote to beta channel
replicated release promote 42 beta --app myapp --version 1.2.3
```

### Promote to Multiple Channels

```bash
# Promote to beta first
replicated release promote 42 beta --app myapp --version 1.2.3

# After testing, promote to stable
replicated release promote 42 stable --app myapp --version 1.2.3
```

## Helm Chart Packaging

Package Helm charts into the manifest directory before creating the release:

```bash
helm package ./chart-dir \
  --destination ./manifests \
  --dependency-update
```

## Inspecting and Downloading Releases

```bash
# List recent releases
replicated release ls --app <app-slug>

# Inspect a specific release
replicated release inspect <sequence> --app <app-slug>

# Download release manifests for audit/debug
replicated release download <sequence> \
  --app <app-slug> \
  --dest ./downloaded-release
```

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Create Replicated Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install Replicated CLI
        run: |
          curl -sSL https://raw.githubusercontent.com/replicatedhq/replicated/main/install.sh | bash
      - name: Create Release
        env:
          REPLICATED_API_TOKEN: ${{ secrets.REPLICATED_API_TOKEN }}
        run: |
          VERSION="${{ github.ref_name }}-${{ github.sha }}"
          replicated release create \
            --app ${{ secrets.REPLICATED_APP_SLUG }} \
            --yaml-dir ./manifests \
            --lint \
            --promote ${{ github.ref_name }} \
            --version $VERSION
```

### Makefile Integration

```makefile
APP_SLUG := myapp
CHANNEL ?= unstable
VERSION ?= $(shell cat VERSION)

.PHONY: release
release: lint package promote

.PHONY: lint
lint:
	@echo "Linting manifests..."
	replicated release lint --app $(APP_SLUG) --yaml-dir manifests

.PHONY: package
package:
	@echo "Packaging Helm charts..."
	rm -f manifests/*.tgz
	helm dependency update ./helm
	helm package ./helm --destination manifests

.PHONY: create
create: package
	@echo "Creating release $(VERSION)..."
	replicated release create \
	  --app $(APP_SLUG) \
	  --yaml-dir manifests \
	  --version $(VERSION)

.PHONY: promote
promote:
	@echo "Promoting to $(CHANNEL)..."
	@[ "$(CHANNEL)" ] || (echo "CHANNEL not set"; exit 1)
	@[ "$(VERSION)" ] || (echo "VERSION not set"; exit 1)
	$(eval SEQUENCE := $(shell replicated release ls --app $(APP_SLUG) | \
	  grep $(VERSION) | awk '{print $$1}' | head -1))
	replicated release promote $(SEQUENCE) $(CHANNEL) \
	  --app $(APP_SLUG) \
	  --version $(VERSION)

.PHONY: release-one-shot
release-one-shot: package
	@[ "$(CHANNEL)" ] || (echo "CHANNEL not set"; exit 1)
	@[ "$(VERSION)" ] || (echo "VERSION not set"; exit 1)
	replicated release create \
	  --app $(APP_SLUG) \
	  --yaml-dir manifests \
	  --lint \
	  --promote $(CHANNEL) \
	  --version $(VERSION)
```

## Rollback

To rollback, promote a previous sequence:

```bash
# List releases to find previous version
replicated release ls --app myapp

# Promote older sequence to channel
replicated release promote 40 stable --app myapp --version 1.2.2
```

Note: This creates a new promotion; it does not delete the problematic release.

## Release Notes

```bash
replicated release create \
  --app myapp \
  --yaml-dir ./manifests \
  --version 1.2.3 \
  --release-notes "$(cat RELEASE_NOTES.md)"
```

## Best Practices

- **Always lint before creating releases**
- Use semantic versioning consistently
- Match versions to git branches or commits for traceability
- Promote to branch-named channels during development
- Include release notes with every release
- Document breaking changes clearly

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "No such file or directory" | Verify `--yaml-dir` path exists |
| "Invalid YAML" | Run `replicated release lint` first |
| "Channel not found" | List channels: `replicated channel ls --app <app>` |
| "App not found" | Verify app slug; check auth with `replicated whoami` |

## Further Reading

- **Vendor Documentation**: https://docs.replicated.com/vendor
- **CLI Reference**: https://docs.replicated.com/reference/vendor-cli
