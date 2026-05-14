---
name: replicated-cmx-clusters
description: This skill should be used when the user asks to "create a CMX cluster", "create an EKS cluster", "create a GKE cluster", "list CMX clusters", "get kubeconfig from CMX", "open a CMX cluster shell", or mentions Replicated Compatibility Matrix cloud clusters, managed Kubernetes testing, or kubectl against CMX.
version: 0.3.0
---

# Replicated CMX Clusters

Create and manage cloud-provider Kubernetes clusters via Replicated's Compatibility Matrix (CMX).

## Overview

**CMX clusters** are fully managed Kubernetes clusters from cloud providers (EKS, GKE, AKS). They are used for testing on existing Kubernetes distributions.

**Less commonly used** than VM clusters for Embedded Cluster testing, but useful for distribution-specific validation.

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

## Creating Cloud Provider Clusters

### EKS Cluster

```bash
replicated cluster create \
  --distribution eks \
  --version 1.32 \
  --instance-type m5.xlarge \
  --nodes 1 \
  --ttl 8h \
  --name my-eks-cluster
```

### GKE Cluster

```bash
replicated cluster create \
  --distribution gke \
  --version 1.33 \
  --instance-type n2-standard-4 \
  --nodes 3 \
  --ttl 8h \
  --name my-gke-cluster
```

### VM-Based Clusters (kind, k3s, etc.)

```bash
replicated cluster create \
  --distribution kind \
  --version 1.32.8 \
  --instance-type r1.medium \
  --disk 100 \
  --ttl 8h \
  --name my-kind-cluster
```

## Monitoring Cluster Status

**Use `--watch` for real-time updates (every 2 seconds):**

```bash
replicated cluster ls --watch
```

**Status progression:**
- `queued` — Initial state after creation
- `assigned` — Resources allocated
- `provisioning` — Kubernetes cluster being built
- `running` — Cluster ready, kubeconfig accessible
- `error` — Creation failed

**Timing:**
- Cloud clusters (EKS/GKE/AKS): 10–15 minutes
- VM-based clusters: 5–10 minutes

## Accessing Clusters

### Get kubeconfig

```bash
# Save to file
replicated cluster kubeconfig <cluster-id> > ~/.kube/config-cmx

# Use with kubectl
export KUBECONFIG=~/.kube/config-cmx
kubectl get nodes
```

### Open Interactive Shell

```bash
replicated cluster shell <cluster-id>
# Now kubectl commands work automatically
kubectl get ns
```

### List Clusters (JSON)

```bash
replicated cluster ls --output json
```

## Cleanup

```bash
# Remove specific cluster
replicated cluster rm <cluster-id>
```

## Best Practices

- Use `--watch` flag to monitor cluster creation in real-time
- Set appropriate TTL values (default 1h, longer for cloud clusters)
- Use `replicated cluster shell <id>` for quick kubectl access without managing kubeconfig
- Cloud clusters support auto-scaling and node groups

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Cluster stuck in `provisioning` | Check Replicated status page; verify quota |
| `kubectl` commands fail | Ensure kubeconfig is exported; try `replicated cluster shell` |

## Further Reading

- **CMX Testing**: https://docs.replicated.com/vendor/testing-how-to
- **CLI Reference**: https://docs.replicated.com/reference/vendor-cli
