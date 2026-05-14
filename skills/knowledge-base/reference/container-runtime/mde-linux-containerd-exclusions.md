---
topic: container-runtime
source: https://learn.microsoft.com/en-us/defender-endpoint/linux-exclusions
created: 2026-04-15
updated: 2026-04-15
tags:
  - microsoft-defender
  - containerd
  - k0s
  - embedded-cluster
  - overlayfs
  - security
---

# MDE on Linux: containerd overlayfs "device or resource busy" — Exclusion Fix

## Summary

Microsoft Defender for Endpoint (MDE) on Linux uses fanotify for real-time file monitoring. When MDE scans files being extracted into a containerd overlayfs tmpmount during an image pull, it holds the mount open. Containerd then cannot unmount the temp directory, producing `device or resource busy` errors. Adding folder and process exclusions for containerd's working paths resolves this without disabling MDE entirely.

## Key Concepts

### How the Failure Manifests

```
failed to pull and unpack image "...":
  failed to extract layer sha256:<digest>:
    failed to unmount /var/lib/embedded-cluster/k0s/containerd/tmpmounts/containerd-mount<N>:
      failed to unmount target .../containerd-mount<N>: device or resource busy: unknown
```

- Each retry creates a *new* randomly-named tmpmount directory; old ones are never cleaned up
- Problem accumulates: 30+ stale mounts can build up over minutes if left running
- Affects larger image layers preferentially — MDE's scan of small layers often completes before containerd's unmount window; larger layers (e.g., kube-proxy) reliably trigger it
- Survives `systemctl restart k0scontroller` (kernel mounts persist across containerd restarts; only full reboot clears them, but MDE immediately reacquires on the next pull attempt)

### Detection Signature in Support Bundles

In the host dmesg / systemd journal output, look for:
```
/usr/lib/systemd/system/mde_netfilter_v2.socket: ListenStream= references ...
```
This confirms MDE is active on the node.

### MDE Exclusion Scopes

| Scope | Flag | Coverage | Notes |
|-------|------|----------|-------|
| `epp` | `--scope epp` | Antivirus only (RTP, BM, on-demand) | EDR alerts still fire |
| `global` | `--scope global` | Antivirus + EDR (RTP, BM, EDR) | No wildcards; path must exist; requires MDE ≥ 101.23092.0012 |

For the containerd tmpmount issue, `epp` scope is sufficient because the problem is real-time scanning (not EDR alerts).

## Practical Application

### Minimum Exclusions (epp scope)

```bash
# Containerd temp mount dir — this is where EBUSY occurs
mdatp exclusion folder add --path /var/lib/containerd/tmpmounts/ --scope epp

# k0s default data dir (standard k0s installs)
mdatp exclusion folder add --path /var/lib/k0s/containerd/tmpmounts/ --scope epp

# Embedded Cluster (EC) k0s data dir
mdatp exclusion folder add --path /var/lib/embedded-cluster/k0s/containerd/tmpmounts/ --scope epp

# Verify
mdatp exclusion list
```

### Broader Exclusions (if minimum is insufficient)

```bash
# Full containerd data dir — covers snapshotters, content store, etc.
mdatp exclusion folder add --path /var/lib/containerd/ --scope epp
mdatp exclusion folder add --path /var/lib/embedded-cluster/k0s/containerd/ --scope epp

# Process exclusion — tell MDE to ignore all file I/O opened by containerd-shim
mdatp exclusion process add --path /var/lib/embedded-cluster/k0s/bin/containerd --scope global
```

### Emergency: Temporarily Disable Real-Time Protection

```bash
mdatp real-time-protection disable
# pull images / run upgrade
mdatp real-time-protection enable
```

### Managed Config (`/etc/opt/microsoft/mdatp/managed/mdatp_managed.json`)

For fleet-managed nodes (Ansible/Puppet/Intune):

```json
{
  "exclusionSettings": {
    "mergePolicy": "merge",
    "exclusions": [
      {
        "$type": "excludedPath",
        "isDirectory": true,
        "path": "/var/lib/containerd/",
        "scopes": ["epp"]
      },
      {
        "$type": "excludedPath",
        "isDirectory": true,
        "path": "/var/lib/embedded-cluster/k0s/containerd/",
        "scopes": ["epp"]
      },
      {
        "$type": "excludedPath",
        "isDirectory": true,
        "path": "/var/lib/k0s/containerd/",
        "scopes": ["epp"]
      }
    ]
  }
}
```

### Diagnosing Which Process Is Causing Events

```bash
# Enable real-time stats
mdatp config real-time-protection-statistics --value enabled

# View top scanning processes
mdatp diagnostic real-time-protection-statistics --output json | python3 -m json.tool
```

## Decision Points

| Situation | Recommendation |
|-----------|---------------|
| Upgrade actively blocked right now | `mdatp real-time-protection disable`, complete upgrade, re-enable |
| Permanent fix on a managed fleet | Use `mdatp_managed.json` via Ansible/Puppet, push to all EC nodes |
| Only some images fail (larger ones) | `tmpmounts/` exclusion alone is sufficient — targets the EBUSY path precisely |
| All image pulls fail consistently | Add full `/var/lib/embedded-cluster/k0s/containerd/` exclusion |
| Security team won't allow folder exclusion | Process exclusion for the containerd binary (`--scope global`) is narrower and may be acceptable |
| EDR alerts are also firing on containerd paths | Switch to `global` scope, requires MDE ≥ 101.23092.0012 |

### Scope of Protection Impact

- `epp` folder exclusion for tmpmounts: very low impact — excludes only short-lived temp extraction dirs
- `epp` for entire containerd dir: moderate — container image layers won't be AV-scanned at rest
- `global` for containerd process: higher — all containerd file I/O excluded from both AV and EDR

## References

- [Configure exclusions for MDE on Linux](https://learn.microsoft.com/en-us/defender-endpoint/linux-exclusions) — official exclusion CLI + managed JSON reference
- [MDE Linux performance troubleshooting](https://learn.microsoft.com/en-us/defender-endpoint/linux-support-perf) — `real-time-protection-statistics` diagnostic workflow
- [containerd#5538: Pulling image fails - failed to unmount temp mount](https://github.com/containerd/containerd/issues/5538) — upstream issue confirming security scanning software as root cause
- Vault note: `[[MDE on Linux - containerd overlayfs exclusions]]`
