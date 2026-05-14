---
name: replicated-cmx-vms
description: This skill should be used when the user asks to "create a CMX VM", "SSH to a CMX VM", "create a multi-node VM cluster", "execute commands on a CMX VM", "expose a VM port", "list CMX VMs", "clean up CMX VMs", or mentions Replicated Compatibility Matrix VMs, VM networking, or VM clusters for Embedded Cluster testing.
version: 0.3.0
---

# Replicated CMX VMs

Create, manage, and operate Replicated Compatibility Matrix (CMX) VMs for testing.

## Overview

**CMX VMs** provide ephemeral infrastructure for testing Kubernetes distributions and applications. VMs on the same network can communicate with each other, making them ideal for multi-node Embedded Cluster testing.

### Instance Types

| Type | vCPU | RAM |
|------|------|-----|
| `r1.medium` | 4 | 8GB |
| `r1.large` | 8 | 16GB |
| `r1.xlarge` | 16 | 32GB |

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

## VM Creation

### Single VM

```bash
replicated vm create \
  --name my-test-vm \
  --distribution ubuntu \
  --version 24.04 \
  --instance-type r1.medium \
  --disk 100 \
  --ttl 8h \
  --wait 5m
```

### Multi-Node Cluster

VMs must be created sequentially: the first VM establishes the network, and subsequent VMs attach to it.

#### 2-Node Cluster (1 Control + 1 Worker)

```bash
# Create control node (establishes network)
replicated vm create \
  --name myapp-control-1 \
  --distribution ubuntu --version 24.04 \
  --instance-type r1.medium \
  --disk 100 --ttl 8h \
  --tag cluster=myapp \
  --wait 5m

# Get network ID
network_id=$(replicated vm ls --output json | \
  jq -r '.[] | select(.name == "myapp-control-1") | .network_id')

# Create worker on same network
replicated vm create \
  --name myapp-worker-1 \
  --distribution ubuntu --version 24.04 \
  --instance-type r1.medium \
  --disk 100 --ttl 8h \
  --network $network_id \
  --tag cluster=myapp \
  --wait 5m
```

#### 3-Node HA Cluster (All Control Plane)

```bash
# Create control-1 (establishes network)
replicated vm create \
  --name myapp-control-1 \
  --distribution ubuntu --version 24.04 \
  --instance-type r1.xlarge \
  --disk 100 --ttl 8h \
  --tag cluster=myapp \
  --wait 5m

# Get network ID
network_id=$(replicated vm ls --output json | \
  jq -r '.[] | select(.name == "myapp-control-1") | .network_id')

# Create control-2 and control-3 in parallel
for i in 2 3; do
  replicated vm create \
    --name myapp-control-$i \
    --distribution ubuntu --version 24.04 \
    --instance-type r1.xlarge \
    --disk 100 --ttl 8h \
    --network $network_id \
    --tag cluster=myapp \
    --wait 5m &
done
wait
```

#### 6-Node Cluster (3 Control + 3 Worker)

```bash
# Create control-1 (establishes network)
replicated vm create \
  --name myapp-control-1 \
  --distribution ubuntu --version 24.04 \
  --instance-type r1.xlarge \
  --disk 100 --ttl 8h \
  --tag role=control --tag cluster=myapp \
  --wait 5m

# Get network ID
network_id=$(replicated vm ls --output json | \
  jq -r '.[] | select(.name == "myapp-control-1") | .network_id')

# Create control-2 and control-3 in parallel
for i in 2 3; do
  replicated vm create \
    --name myapp-control-$i \
    --distribution ubuntu --version 24.04 \
    --instance-type r1.xlarge \
    --disk 100 --ttl 8h \
    --network $network_id \
    --tag role=control --tag cluster=myapp \
    --wait 5m &
done

# Create workers 1-3 in parallel
for i in 1 2 3; do
  replicated vm create \
    --name myapp-worker-$i \
    --distribution ubuntu --version 24.04 \
    --instance-type r1.xlarge \
    --disk 100 --ttl 8h \
    --network $network_id \
    --tag role=worker --tag cluster=myapp \
    --wait 5m &
done
wait
```

## Monitoring VM Status

**Use `--watch` for real-time updates (every 2 seconds):**

```bash
replicated vm ls --watch
```

VMs typically reach `running` status within 2–3 minutes.

## SSH Access and Command Execution

### Interactive SSH

```bash
replicated vm ssh <vm-id-or-name>
```

### Remote Command Execution

```bash
# Get VM ID
vm_id=$(replicated vm ls --output json | \
  jq -r '.[] | select(.name == "myapp-control-1") | .id')

# Execute command
replicated vm ssh $vm_id -- 'kubectl get nodes'

# With sudo
replicated vm ssh $vm_id -- 'sudo systemctl status k0scontroller'

# Multi-line command
replicated vm ssh $vm_id -- 'kubectl get nodes && kubectl get pods -A'
```

### Parallel Command Execution

```bash
# Get all control node IDs
control_vms=$(replicated vm ls --output json | \
  jq -r '.[] | select(.tags.cluster == "myapp" and .tags.role == "control") | .id')

# Execute on all control nodes in parallel
for vm_id in $control_vms; do
  replicated vm ssh $vm_id -- 'kubectl get nodes' &
done
wait
```

### File Transfer

**Download from VM:**
```bash
replicated vm ssh $vm_id -- \
  'sudo cat /var/lib/embedded-cluster/k0s/pki/admin.conf' > kubeconfig.yaml
```

**Upload to VM:**
```bash
cat local-file.yaml | \
  replicated vm ssh $vm_id -- 'cat > /tmp/remote-file.yaml'
```

## Port Exposure

Expose VM ports for external access with automatic DNS and TLS:

```bash
# Expose port
replicated vm port expose <vm-id-or-name> --port 8080 --protocol http

# Expose HTTPS port
replicated vm port expose <vm-id-or-name> --port 443 --protocol https

# List exposed ports with DNS endpoints
replicated vm port ls <vm-id-or-name>

# Remove exposed port
replicated vm port rm <vm-id-or-name> --port 8080
```

**Supported protocols:** `http`, `https`, `ws`, `wss`

**Key features:**
- DNS records automatically created (e.g., `abc123.cmx.replicated.com`)
- TLS certificates automatically provisioned
- Use for accessing Admin Console, application endpoints, etc.

## VM Lifecycle Management

### Extend TTL

Via vendor portal:
1. Navigate to Compatibility Matrix
2. Find VM in list
3. Click "Extend TTL"
4. Set new expiration time

### Stop and Start

```bash
replicated vm stop <vm-id>
replicated vm start <vm-id>
replicated vm restart <vm-id>
```

### Delete VMs

```bash
# Delete single VM
replicated vm rm <vm-id>

# Delete multiple VMs
replicated vm rm <vm-id-1> <vm-id-2>

# Delete all VMs with a tag
replicated vm ls --output json | \
  jq -r '.[] | select(.tags.cluster == "myapp") | .id' | \
  xargs replicated vm rm
```

## Best Practices

- Use `--watch` flag to monitor VM creation (ready in 2–3 minutes)
- Use cluster prefixes (tags) to namespace resources and avoid collisions
- Create first VM, get network ID, then create remaining VMs on same network
- Use parallel VM creation (background with `&`) for faster multi-node setup
- Use `r1.xlarge` for HA clusters, `r1.medium` for dev/test
- Clean up VMs when done to control costs
- Use extended timeouts for slow operations (package installs, downloads)
- Poll for completion when waiting on async operations

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Quota exceeded" | List all VMs: `replicated vm ls`; delete unused VMs |
| Network creation failures | Verify first node completes before creating additional nodes |
| SSH connection timeout | Wait 2–3 minutes after VM creation; verify status is `running` |
| Permission denied | Verify SSH key configuration; use `--ssh-public-key-github <user>` |
| Cannot access exposed ports | Verify port is exposed; check firewall rules on VM |

## Further Reading

- **CMX Testing**: https://docs.replicated.com/vendor/testing-how-to
- **CLI Reference**: https://docs.replicated.com/reference/vendor-cli
