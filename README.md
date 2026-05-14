# OpenCode Skills — Replicated CLI

Action-oriented [OpenCode](https://github.com/opencode-ai/opencode) skills for the [Replicated CLI](https://docs.replicated.com/reference/vendor-cli).

## Philosophy

These skills are organized by **common action**, not by product, following the pattern established by [DataDog's pup CLI](https://github.com/DataDog/pup/tree/main/skills). Each skill has a narrow description so OpenCode invokes the right guidance for the user's intent.

## Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `replicated-releases` | "create a release", "promote a release", "lint manifests", "package a Helm chart" | Release lifecycle, versioning, CI/CD, Makefiles |
| `replicated-channels` | "list channels", "create a channel", "channel strategy" | Channel management, deployment tracks, promotion workflows |
| `replicated-customers` | "list customers", "create a customer", "download a license" | Customer management, licenses, entitlements, instances |
| `replicated-cmx-clusters` | "create a CMX cluster", "EKS cluster", "get kubeconfig" | Cloud-provider Kubernetes (EKS, GKE, AKS, kind, k3s) |
| `replicated-cmx-vms` | "create a CMX VM", "SSH to a VM", "multi-node cluster", "expose a port" | VM creation, multi-node networking, SSH, remote execution, cleanup |
| `replicated-embedded-cluster` | "install Embedded Cluster", "join a node", "EC status" | EC binary download, installation, node joining, verification |
| `replicated-api` | "make an API call", "Vendor API", "query the API" | Ad-hoc GET/POST/PUT/PATCH to the Vendor API v3, `jq` filtering, scripting |

## Authentication

All skills assume Replicated CLI authentication via `replicated login` or environment variables. Projects typically use `direnv` with an `.envrc` at the repo root:

```bash
export REPLICATED_API_TOKEN="your-api-token-here"
export REPLICATED_APP="app-slug"
```

## Installation

Copy the desired skill directories into your OpenCode skills path:

```bash
# e.g. ~/.config/opencode/skills/
cp -r skills/* ~/.config/opencode/skills/
```

## Further Reading

- [Replicated Vendor Documentation](https://docs.replicated.com/vendor)
- [Replicated CLI Reference](https://docs.replicated.com/reference/vendor-cli)
