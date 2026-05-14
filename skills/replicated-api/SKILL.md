---
name: replicated-api
description: This skill should be used when the user asks to "make a Replicated API call", "use the Vendor API", "call the Replicated API", "replicated api get/post/put/patch", "query the Replicated API", or mentions ad-hoc Replicated API requests, Vendor API v3, or programmatic access to the Vendor Portal.
version: 0.1.0
---

# Replicated API

Make ad-hoc HTTP calls to the Replicated Vendor API v3 using the `replicated api` subcommand.

## Overview

The `replicated api` subcommand is a built-in `curl`-like client for the Replicated Vendor API. It automatically handles authentication using your local CLI credentials and prints the raw JSON response.

**Base URL:** `https://api.replicated.com/vendor`

**API version:** `v3`

**Documentation:** https://replicated-vendor-api.readme.io/v3/reference/
**Enterprise docs:** https://docs.replicated.com/reference/vendor-api-using
**Swagger spec:** https://api.replicated.com/vendor/v3/spec/vendor-api-v3.json

## Authentication

The `replicated api` command uses the same authentication as the rest of the CLI (via `replicated login` or `--token`). No extra headers are required.

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
- This eliminates the need for `--app` and `--token` flags in most commands

## Available Commands

| Command | Purpose | Has `--body` flag |
|---------|---------|-------------------|
| `replicated api get` | GET request | No |
| `replicated api post` | POST request | Yes (`-b`) |
| `replicated api put` | PUT request | Yes (`-b`) |
| `replicated api patch` | PATCH request | Yes (`-b`) |

## Usage Pattern

Pass the **path** (without host or version) as the final argument. The CLI prepends `https://api.replicated.com/vendor`.

**Recommended:** Pipe output to `jq` for readable JSON.

## GET Requests

```bash
# List apps
replicated api get /v3/apps | jq

# Get a specific app
replicated api get /v3/app/<app-id-or-slug> | jq

# List channels for an app
replicated api get /v3/app/<app-id>/channels | jq

# List customers
replicated api get /v3/customers | jq

# Get a specific customer
replicated api get /v3/customer/<customer-id> | jq

# List releases
replicated api get /v3/app/<app-id>/releases | jq

# Get a specific release
replicated api get /v3/app/<app-id>/release/<sequence> | jq
```

## POST Requests

```bash
# Create a channel
replicated api post /v3/app/<app-id>/channel \
  -b '{"name":"beta","description":"Beta testing"}' | jq

# Create a customer
replicated api post /v3/app/<app-id>/customer \
  -b '{"name":"Acme Corp","email":"admin@acme.com","channel_id":"<channel-id>"}' | jq
```

## PUT Requests

```bash
# Update a channel
replicated api put /v3/app/<app-id>/channel/<channel-id> \
  -b '{"name":"stable","description":"Production releases"}' | jq

# Update a customer
replicated api put /v3/customer/<customer-id> \
  -b '{"name":"Acme Corp Updated","email":"new@acme.com"}' | jq
```

## PATCH Requests

```bash
# Partially update a customer name
replicated api patch /v3/customer/<customer-id> \
  -b '{"name":"Valuable Customer"}' | jq
```

## Common API Paths

| Resource | Path |
|----------|------|
| Apps | `/v3/apps` |
| App details | `/v3/app/<app-id>` |
| Channels | `/v3/app/<app-id>/channels` |
| Channel | `/v3/app/<app-id>/channel/<channel-id>` |
| Releases | `/v3/app/<app-id>/releases` |
| Release | `/v3/app/<app-id>/release/<sequence>` |
| Customers | `/v3/customers` |
| Customer | `/v3/customer/<customer-id>` |
| Instances | `/v3/app/<app-id>/instances` |

## Working with `--app` Slug

When the `--app` global flag is set (or `REPLICATED_APP` env var is loaded), some API endpoints resolve the app ID automatically.

```bash
# With REPLICATED_APP set, some endpoints work by slug
replicated api get /v3/app/myapp/releases | jq
```

## jq Filtering Patterns

```bash
# Extract just channel names
replicated api get /v3/app/<app-id>/channels | jq -r '.channels[].name'

# Find customer by email
replicated api get /v3/customers | jq '.customers[] | select(.email == "admin@acme.com")'

# Get latest release sequence
replicated api get /v3/app/<app-id>/releases | jq '.releases | sort_by(.sequence) | last | .sequence'

# Pretty-print with colors
replicated api get /v3/apps | jq .
```

## Scripting Patterns

### Create and Promote Release via API

```bash
#!/bin/bash
set -e

APP_ID="myapp"
CHANNEL_ID="stable"
VERSION="1.2.3"

# Create release
release=$(replicated api post /v3/app/$APP_ID/release \
  -b "{\"yaml\":\"$(base64 -i manifests/*.yaml | tr '\n' ',' )\",\"version\":\"$VERSION\"}")
sequence=$(echo "$release" | jq -r '.release.sequence')

# Promote to channel
replicated api post /v3/app/$APP_ID/release/$sequence/promote \
  -b "{\"channel_ids\":[\"$CHANNEL_ID\"],\"version\":\"$VERSION\"}"
```

### Bulk Customer Operations

```bash
# Get all customers
replicated api get /v3/customers | jq '.customers' > customers.json

# Update expiration for a subset
for id in $(jq -r '.[] | select(.license_type == "trial") | .id' customers.json); do
  replicated api patch /v3/customer/$id \
    -b '{"expires_at":"2026-12-31T00:00:00Z"}'
done
```

## Error Handling

The CLI prints the raw HTTP response body on error. Common patterns:

```bash
# Check status code inline
response=$(replicated api get /v3/app/nonexistent 2>&1)
if echo "$response" | grep -q '"error"'; then
  echo "Request failed: $response"
fi
```

## Best Practices

- **Always pipe to `jq`** for readable output and structured filtering
- Use `--app` or `REPLICATED_APP` to avoid passing app IDs in every path
- Store API responses in variables when chaining multiple calls
- Use `replicated api get` to inspect state before making mutating calls
- Prefer the high-level CLI commands (`replicated release create`, `replicated channel create`, etc.) when available; use `replicated api` for operations not yet exposed in the CLI

## Further Reading

- **Vendor API v3 Reference:** https://replicated-vendor-api.readme.io/v3/reference/
- **Using the Vendor API:** https://docs.replicated.com/reference/vendor-api-using
- **Swagger spec:** https://api.replicated.com/vendor/v3/spec/vendor-api-v3.json
