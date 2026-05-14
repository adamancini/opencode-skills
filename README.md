# OpenCode Skills — DevOps Toolkit

Action-oriented [OpenCode](https://github.com/opencode-ai/opencode) skills for DevOps, SRE, and platform engineering workflows.

## Philosophy

These skills are organized by **common action**, not by product, following the pattern established by [DataDog's pup CLI](https://github.com/DataDog/pup/tree/main/skills). Each skill has a narrow description so OpenCode invokes the right guidance for the user's intent.

## Replicated Platform Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `replicated-releases` | "create a release", "promote a release", "lint manifests", "package a Helm chart" | Release lifecycle, versioning, CI/CD, Makefiles |
| `replicated-channels` | "list channels", "create a channel", "channel strategy" | Channel management, deployment tracks, promotion workflows |
| `replicated-customers` | "list customers", "create a customer", "download a license" | Customer management, licenses, entitlements, instances |
| `replicated-cmx-clusters` | "create a CMX cluster", "EKS cluster", "get kubeconfig" | Cloud-provider Kubernetes (EKS, GKE, AKS, kind, k3s) |
| `replicated-cmx-vms` | "create a CMX VM", "SSH to a VM", "multi-node cluster", "expose a port" | VM creation, multi-node networking, SSH, remote execution, cleanup |
| `replicated-embedded-cluster` | "install Embedded Cluster", "join a node", "EC status" | EC binary download, installation, node joining, verification |
| `replicated-api` | "make an API call", "Vendor API", "query the API" | Ad-hoc GET/POST/PUT/PATCH to the Vendor API v3, `jq` filtering, scripting |

## Kubernetes & Helm Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `helm-chart-developer` | "create a Helm chart", "review Helm chart", "Helm best practices" | Helm 3 standards, templating, values.yaml, RBAC, production readiness |
| `yaml-kubernetes-validator` | "validate YAML", "check Kubernetes manifest", "review YAML" | yaml-language-server compliance, K8s API specs, deprecated APIs |
| `cluster-context-manager` | "switch context", "merge kubeconfig", "list clusters" | kubectl/talosctl context management, kubecm, kubeconfig operations |

## Infrastructure & Automation Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `proxmox-manager` | "spin up staging", "Proxmox", "Talos", "upgrade Kubernetes" | Proxmox VE clusters, VM lifecycle, Talos Linux, Ansible, Taskfile workflows |
| `ansible-playbook-guide` | "write Ansible playbook", "Ansible role", "inventory" | Playbook development, modules, roles, Vault secrets, Jinja2 |
| `ssl-cert-manager` | "create certificate", "Let's Encrypt", "TLS secret" | SSL/TLS certs, ACME, DNS challenges, K8s TLS secrets |
| `system-updates` | "update my system", "update brew", "run updates" | Coordinated Homebrew, yadm, and pass updates |

## Development & Workflow Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `git-worktree-manager` | "create worktree", "parallel development", "gwt" | Git worktrees, `.worktrees/` convention, agent isolation |
| `git-repo-organizer` | "clone repo", "where to place", "organize repos" | GOPATH conventions, Go vs non-Go repository placement |
| `shell-code-optimizer` | "write shell script", "review bash", "POSIX compliance" | bash/zsh portability, shellcheck, built-ins over external commands |
| `zsh-config-manager` | "add shell config", "zsh completions", "conf.d" | Modular zsh setup, yadm alternates, OS-specific configs |
| `aerospace-config-manager` | "configure aerospace", "keybinding", "workspace" | AeroSpace window manager, TOML, macOS tiling |
| `markdown-writer` | "write markdown", "markdownlint", "README" | MD001-MD048 compliance, technical docs, API documentation |
| `web-search-researcher` | "research", "look up", "find documentation" | Structured web research, multi-query methodology, source synthesis |

## Quality & Security Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `mcp-security-validator` | "validate MCP server", "check security", "safe to install" | Backslash Security Hub, Snyk Advisor, blocklist, risk scoring |
| `claudemd-compliance-checker` | "check compliance", "audit code", "CLAUDE.md rules" | Project instruction compliance, PASS/FAIL/REVIEW methodology |
| `quality-control-enforcer` | "review implementation", "quality check", "root cause" | Detect workarounds, simulated data, incomplete implementations |

## Knowledge Management Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `obsidian-notes` | "create note", "vault organization", "Obsidian" | Vault structure, frontmatter, MOCs, knowledge ingestion |
| `knowledge-base` | "what do I know about", "reference material", "curated" | Curated reference library, machine-optimized technical knowledge |
| `knowledge-ingest` | "learn this", "teach me this", "distill article" | Automated URL ingestion, vault + knowledge-base dual storage |
| `knowledge-reader` | "search my notes", "find in vault", "what do I know" | RAG layer over personal knowledge graph, multi-source synthesis |
| `notion-sync` | "sync to Notion", "push notes", "Notion integration" | Obsidian-to-Notion sync, work note exclusion for privacy |

## Tool & Integration Skills

| Skill | Trigger | What it covers |
|-------|---------|----------------|
| `yadm-utilities` | "track with yadm", "dotfiles", "bootstrap" | YADM dotfiles manager, alternates, bootstrap automation |
| `linear-mcp-operations` | "Linear issue", "create ticket", "project cycle" | Linear MCP health checks, validated operations, response verification |
| `friction-log` | "this is confusing", "blocker", "pain point" | Structured friction capture, bootcamp feedback, developer experience |

## Authentication

Replicated platform skills assume Replicated CLI authentication via `replicated login` or environment variables. Projects typically use `direnv` with an `.envrc` at the repo root:

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

Or install via the [Superpowers plugin](https://github.com/obra/superpowers):

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

## Further Reading

- [Replicated Vendor Documentation](https://docs.replicated.com/vendor)
- [Replicated CLI Reference](https://docs.replicated.com/reference/vendor-cli)
- [OpenCode Skills Documentation](https://github.com/opencode-ai/opencode)
