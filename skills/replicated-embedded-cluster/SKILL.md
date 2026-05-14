---
name: replicated-embedded-cluster
description: This skill should be used when the user asks to "install Embedded Cluster", "join a node to Embedded Cluster", "check Embedded Cluster status", "download the Embedded Cluster binary", or mentions Replicated Embedded Cluster, EC installation, or k0s controller status on CMX VMs.
version: 0.3.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,embedded-cluster,ec,k0s
  globs: ""
  alwaysApply: "false"
---

# Replicated Embedded Cluster

Install and manage Replicated Embedded Cluster on CMX VMs.

## Overview

**Embedded Cluster (EC)** is Replicated's Kubernetes distribution that bundles the cluster, KOTS admin console, and application into a single installer binary.

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

## Installation Workflow

### 1. Create VM Cluster

See `replicated-cmx-vms` skill for VM creation. Typical setup:

```bash
# Create 3 VMs on the same network
# (control-1 establishes network, then control-2 and control-3 join it)
```

### 2. Download EC Binary

```bash
# SSH to the first node
replicated vm ssh myapp-control-1

# Download and extract the EC binary
curl -f "https://app.slug.replicated.com/embedded/<channel>" \
  -H "Authorization: <license-id>" \
  -o app.tgz
tar -xzf app.tgz
```

### 3. Install on First Node

```bash
sudo ./app install \
  --license <license-file> \
  --admin-console-password <password>
```

**Installation takes 10–15 minutes.**

### 4. Monitor Installation Status

```bash
sudo ./app status
```

### 5. Generate Join Command for Additional Nodes

```bash
sudo ./app join print-command
```

### 6. Join Remaining Nodes

On each additional node:

```bash
sudo ./app join <join-command> --yes
```

## Post-Installation Verification

### Check Nodes

```bash
kubectl get nodes
```

### Check System Pods

```bash
kubectl get pods -n kube-system
```

### Retrieve kubeconfig (from local machine)

```bash
vm_id=$(replicated vm ls --output json | \
  jq -r '.[] | select(.name == "myapp-control-1") | .id')

replicated vm ssh $vm_id -- \
  'sudo cat /var/lib/embedded-cluster/k0s/pki/admin.conf' > kubeconfig.yaml

export KUBECONFIG=kubeconfig.yaml
kubectl get nodes
```

### Check Controller Logs

```bash
replicated vm ssh <vm-id> -- \
  'sudo journalctl -u k0scontroller --no-pager'
```

## Best Practices

- Ensure VMs have 100GB+ disk space
- Verify VMs have internet connectivity for downloads
- Use `r1.xlarge` instance type for production-like HA clusters
- Use `--yes` flags for non-interactive installations in automation
- Only run one installation at a time per node

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Installation hangs | Check disk space; verify internet connectivity |
| Nodes not joining | Verify network connectivity between VMs; check join command |
| Controller not starting | Review logs: `sudo journalctl -u k0scontroller` |
| Admin Console unreachable | Verify ports are exposed; check VM firewall rules |

## Further Reading

- **Embedded Cluster Overview**: https://docs.replicated.com/vendor/embedded-overview
- **CLI Reference**: https://docs.replicated.com/reference/vendor-cli
