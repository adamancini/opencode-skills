---
name: replicated-cmx-networks
description: This skill should be used when the user asks to "list CMX networks", "manage Replicated networks", "network ls", "network report", "network update", or mentions Replicated Compatibility Matrix networks, VM networks, or cluster networks.
version: 0.1.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: replicated,cmx,network,vm,cluster
  globs: ""
  alwaysApply: "false"
---

# Replicated CMX Networks

Manage test networks for VMs and clusters in the Replicated Compatibility Matrix.

## Overview

**Networks** are used by CMX VMs and clusters to communicate with each other. When you create a VM cluster, the first VM establishes the network, and subsequent VMs attach to it.

## Commands

### List Networks

```bash
# List all networks
replicated network ls

# List with JSON output
replicated network ls --output json

# Watch for changes
replicated network ls --watch
```

### Update Network

```bash
# Update a network with an airgap policy
replicated network update <network-id> --policy airgap
```

### Get Network Report

```bash
replicated network report <network-id>
```

## Best Practices

- Networks are automatically created when the first VM in a cluster is provisioned
- Use `--watch` to monitor network status in real-time
- Apply airgap policies for testing offline/airgapped scenarios

## Further Reading

- **Replicated CLI Network**: https://docs.replicated.com/reference/replicated-cli-network
