---
name: replicated-channels
description: This skill should be used when the user asks to "list Replicated channels", "create a channel", "inspect a channel", "promote a release to a channel", or mentions Replicated channel management, deployment tracks, or channel strategy (stable, beta, unstable).
version: 0.3.0
---

# Replicated Channels

Manage deployment tracks (channels) that customers subscribe to for updates.

## Overview

**Channels** represent deployment tracks (e.g., `stable`, `beta`, `unstable`) that customers subscribe to. Releases are promoted to channels to make them available.

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

## Channel Commands

### List Channels

```bash
replicated channel ls --app <app-slug>
```

### Create a Channel

```bash
replicated channel create \
  --app <app-slug> \
  --name <channel-name> \
  --description "Channel description"
```

### Inspect a Channel

```bash
replicated channel inspect <channel-id> --app <app-slug>
```

### List Releases on a Channel

```bash
replicated channel releases <channel-id> --app <app-slug>
```

## Channel Strategy

### Development Workflow

- Use branch-named channels during development (e.g., `feature-auth`, `bugfix-123`)
- Automatically create channels matching git branches
- Clean up channels when branches are merged/deleted

### Release Workflow

| Channel | Purpose |
|---------|---------|
| `unstable` | Latest builds, internal testing only |
| `beta` | Pre-release testing with select customers |
| `stable` | Production-ready releases |

## Promoting Releases to Channels

### Promote by Sequence Number

```bash
# Create release (note the sequence from output)
replicated release create --app myapp --yaml-dir ./manifests --version 1.2.3
# Output: SEQUENCE: 42

# Promote to beta channel
replicated release promote 42 beta --app myapp --version 1.2.3
```

### Promote to Multiple Channels

```bash
# Beta first
replicated release promote 42 beta --app myapp --version 1.2.3

# Then stable after testing
replicated release promote 42 stable --app myapp --version 1.2.3
```

## Verifying Channel Promotion

```bash
# Check which channels have a specific release
replicated release inspect <sequence> --app <app-slug> | grep -i channel

# List releases on a channel
replicated channel releases <channel-id> --app <app-slug>
```

## Best Practices

- Maintain separate channels for testing stages (`unstable`, `beta`, `stable`)
- Use branch-named channels during development
- Clean up obsolete channels
- Document channel promotion criteria
- Match release versions to git tags or commits for traceability

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Channel not found" | List channels: `replicated channel ls --app <app>`; check spelling |
| "App not found" | Verify app slug; check auth with `replicated whoami` |

## Further Reading

- **Vendor Documentation**: https://docs.replicated.com/vendor
- **CLI Reference**: https://docs.replicated.com/reference/vendor-cli
