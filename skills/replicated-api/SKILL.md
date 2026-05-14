---
name: replicated-api
description: Make ad-hoc GET/POST/PUT/PATCH calls to the Replicated Vendor API v3. Use when the user asks to "call the Replicated API", "use the Vendor API", "replicated api get/post/put/patch", or mentions programmatic access to the Vendor Portal.
version: 0.1.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,vendor-api,api,curl,jq
  globs: ""
  alwaysApply: "false"
---

# Replicated API

Make ad-hoc HTTP calls to the Replicated Vendor API v3.

## Overview

`replicated api` is a built-in `curl`-like client for the Vendor API. It handles auth automatically and prints raw JSON.

- **Base URL:** `https://api.replicated.com/vendor`
- **Version:** `v3`
- **Docs:** https://replicated-vendor-api.readme.io/v3/reference/
- **Enterprise docs:** https://docs.replicated.com/reference/vendor-api-using
- **Swagger:** https://api.replicated.com/vendor/v3/spec/vendor-api-v3.json

## Authentication

Uses the same auth as the rest of the CLI:

```bash
replicated login
```

API tokens: https://vendor.replicated.com/team/tokens

### Environment Variables (direnv)

```bash
export REPLICATED_API_TOKEN="your-api-token-here"
export REPLICATED_APP="app-slug"
```

## Commands

| Command | Purpose | Body flag |
|---------|---------|-----------|
| `api get` | GET request | — |
| `api post` | POST request | `-b` |
| `api put` | PUT request | `-b` |
| `api patch` | PATCH request | `-b` |

Pass the **path** (no host or version) as the final argument.

## Examples

### GET

```bash
# List apps
replicated api get /v3/apps | jq

# Get app details
replicated api get /v3/app/<app-id> | jq

# List channels
replicated api get /v3/app/<app-id>/channels | jq

# List customers
replicated api get /v3/customers | jq

# List releases
replicated api get /v3/app/<app-id>/releases | jq
```

### POST

```bash
# Create a channel
replicated api post /v3/app/<app-id>/channel \
  -b '{"name":"beta","description":"Beta testing"}' | jq

# Create a customer
replicated api post /v3/app/<app-id>/customer \
  -b '{"name":"Acme Corp","email":"admin@acme.com","channel_id":"<id>"}' | jq
```

### PUT

```bash
# Update a channel
replicated api put /v3/app/<app-id>/channel/<channel-id> \
  -b '{"name":"stable","description":"Production releases"}' | jq
```

### PATCH

```bash
# Partially update a customer
replicated api patch /v3/customer/<customer-id> \
  -b '{"name":"Valuable Customer"}' | jq
```

## Common API Paths

| Resource | Path |
|----------|------|
| Apps | `/v3/apps` |
| App | `/v3/app/<app-id>` |
| Channels | `/v3/app/<app-id>/channels` |
| Channel | `/v3/app/<app-id>/channel/<channel-id>` |
| Releases | `/v3/app/<app-id>/releases` |
| Release | `/v3/app/<app-id>/release/<sequence>` |
| Customers | `/v3/customers` |
| Customer | `/v3/customer/<customer-id>` |
| Instances | `/v3/app/<app-id>/instances` |

## jq Filtering

```bash
# Extract channel names
replicated api get /v3/app/<app-id>/channels | jq -r '.channels[].name'

# Find customer by email
replicated api get /v3/customers | jq '.customers[] | select(.email == "admin@acme.com")'

# Latest release sequence
replicated api get /v3/app/<app-id>/releases | jq '.releases | sort_by(.sequence) | last | .sequence'
```

## Scripting Pattern

```bash
# Create release and promote via API
release=$(replicated api post /v3/app/$APP_ID/release \
  -b "{\"yaml\":\"$(base64 -i manifests/*.yaml | tr '\n' ',' )\",\"version\":\"$VERSION\"}")
sequence=$(echo "$release" | jq -r '.release.sequence')

replicated api post /v3/app/$APP_ID/release/$sequence/promote \
  -b "{\"channel_ids\":[\"$CHANNEL_ID\"],\"version\":\"$VERSION\"}"
```

## Best Practices

- **Pipe to `jq`** for readable output
- Use `--app` or `REPLICATED_APP` to avoid passing app IDs in every path
- Prefer high-level CLI commands when available; use `replicated api` for operations not yet exposed

## Further Reading

- **API Reference:** https://replicated-vendor-api.readme.io/v3/reference/
- **Using the API:** https://docs.replicated.com/reference/vendor-api-using
- **Swagger spec:** https://api.replicated.com/vendor/v3/spec/vendor-api-v3.json
