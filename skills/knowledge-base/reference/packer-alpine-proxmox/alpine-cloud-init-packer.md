---
topic: packer-alpine-proxmox
source:
  - https://twdev.blog/2023/11/alpine_cloudinit/
  - https://github.com/red-lichtie/alpine-cloud-init
  - https://www.techtutorials.tv/sections/promox/proxmox-alpine-cloud-init-image/
  - https://www.uncommonengineer.com/docs/engineer/LAB/proxmox-packer-vm/
created: 2026-03-12
updated: 2026-03-12
tags:
  - alpine
  - cloud-init
  - packer
  - proxmox
  - templates
  - provisioning
---

# Alpine Linux Cloud-Init Templates for Proxmox with Packer

## Summary

Alpine Linux requires manual cloud-init setup because its installer (`setup-alpine`) has no native cloud-init integration (unlike Ubuntu's autoinstall or Fedora's kickstart). The workflow is: Packer boots the Alpine ISO, drives `setup-alpine` via keystrokes, SSHes in to install cloud-init and dependencies, configures the NoCloud datasource, cleans state, and converts to a Proxmox template. The resulting template accepts standard Proxmox cloud-init parameters (user, SSH keys, IP config) on clone.

## Key Concepts

### Why Alpine Needs Special Handling

- `setup-alpine` is interactive with no cloud-init hooks
- Alpine uses BusyBox by default -- BusyBox `mount` cannot handle the CIDATA ISO that Proxmox generates
- The `cloud-init` APK package omits the `py3-netifaces` dependency
- Alpine uses OpenRC, not systemd -- cloud-init service names differ
- The `setup-cloud-init` helper script exists but must run AFTER manual package installation

### Required Packages

These packages are **mandatory** for cloud-init to function on Alpine with Proxmox:

```bash
apk add \
  cloud-init \
  util-linux \
  e2fsprogs-extra \
  qemu-guest-agent \
  sudo \
  openssh-server-pam

# Optional but recommended
apk add \
  py3-netifaces \
  chrony \
  doas \
  parted \
  lsblk \
  tzdata
```

**Package rationale:**

| Package | Why Required |
|---------|-------------|
| `cloud-init` | Core provisioning framework |
| `util-linux` | Non-BusyBox `mount` -- without this, cloud-init cannot mount the CIDATA ISO |
| `e2fsprogs-extra` | Provides `resize2fs` for cloud-init disk resize on first boot |
| `qemu-guest-agent` | Proxmox VM monitoring, graceful shutdown, IP reporting |
| `sudo` | Cloud-init user provisioning expects sudo for privilege escalation |
| `openssh-server-pam` | Prevents SSH login failures with cloud-init provisioned users |
| `py3-netifaces` | Network interface detection -- not pulled in automatically by Alpine's cloud-init package |

### Cloud-Init Configuration

#### Datasource Lock-Down

Force NoCloud only (prevents slow timeouts probing AWS, GCE, etc.):

```bash
echo "datasource_list: [ NoCloud, ConfigDrive ]" > /etc/cloud/cloud.cfg.d/99_pve.cfg
```

#### Disable Broken Modules

The IBM `reset_rmc` module causes warnings on non-IBM hardware:

```bash
sed -i 's/  - reset_rmc/#  - reset_rmc/' /etc/cloud/cloud.cfg
```

#### Default User Configuration

Override cloud-init's default user group assignments (Alpine doesn't have `adm`, `lxd`, etc.):

```yaml
# /etc/cloud/cloud.cfg.d/03-user.cfg
system_info:
  default_user:
    groups: []
    sudo: ["ALL=(ALL) NOPASSWD: ALL"]
```

Adjust sudo policy per security requirements. For password-required sudo:
```yaml
    sudo: ["ALL=(ALL) ALL"]
```

### The setup-cloud-init Helper

Alpine provides `setup-cloud-init` which:
1. Enables cloud-init OpenRC services (`cloud-init-local`, `cloud-init`, `cloud-config`, `cloud-final`)
2. Configures basic cloud-init integration

**Run this AFTER installing all packages and configuring datasource files.** It is a one-way operation -- take a snapshot before running if building manually.

### OpenRC Service Enablement

If not using `setup-cloud-init`, enable services manually:

```bash
rc-update add cloud-init-local boot
rc-update add cloud-init default
rc-update add cloud-config default
rc-update add cloud-final default
rc-update add qemu-guest-agent
```

## Packer Build

### Project Structure

```
alpine-cloud-init/
  alpine.pkr.hcl              # Main build definition
  variables.pkr.hcl           # Variable declarations
  vars/
    local.pkrvars.hcl          # Environment-specific values (gitignored)
  scripts/
    setup-cloud-init.sh        # Post-install provisioning script
  http/                        # (empty -- Alpine doesn't use Packer HTTP server)
```

### Packer HCL: Variables

```hcl
# variables.pkr.hcl

variable "proxmox_api_url" {
  type = string
}

variable "proxmox_api_token_id" {
  type = string
}

variable "proxmox_api_token_secret" {
  type      = string
  sensitive = true
}

variable "proxmox_node" {
  type    = string
  default = "pve01"
}

variable "storage_pool" {
  type    = string
  default = "local-lvm"
}

variable "alpine_iso" {
  type        = string
  description = "ISO file path on Proxmox storage (e.g., local:iso/alpine-virt-3.21.3-x86_64.iso)"
}

variable "vm_id" {
  type    = number
  default = 9010
}

variable "template_name" {
  type    = string
  default = "alpine-cloud-init"
}

variable "alpine_mirror" {
  type    = string
  default = "dl-cdn.alpinelinux.org"
  description = "Alpine APK mirror hostname (no https:// prefix)"
}

variable "root_password" {
  type      = string
  default   = "packer"
  sensitive = true
  description = "Temporary root password during build (removed before templating)"
}

variable "disk_size" {
  type    = string
  default = "2G"
}

variable "memory" {
  type    = number
  default = 1024
}
```

### Packer HCL: Source Block

```hcl
# alpine.pkr.hcl

packer {
  required_plugins {
    proxmox = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}

source "proxmox-iso" "alpine" {
  # Proxmox connection
  proxmox_url              = var.proxmox_api_url
  username                 = var.proxmox_api_token_id
  token                    = var.proxmox_api_token_secret
  node                     = var.proxmox_node
  insecure_skip_tls_verify = true

  # VM settings
  vm_id    = var.vm_id
  vm_name  = var.template_name
  memory   = var.memory
  cores    = 2
  cpu_type = "host"
  os       = "l26"
  qemu_agent = true

  # Storage
  scsi_controller = "virtio-scsi-single"
  disks {
    type         = "scsi"
    disk_size    = var.disk_size
    storage_pool = var.storage_pool
  }

  # Network
  network_adapters {
    model  = "virtio"
    bridge = "vmbr0"
  }

  # ISO
  iso_file = var.alpine_iso

  # Cloud-init drive (added before template conversion)
  cloud_init              = true
  cloud_init_storage_pool = var.storage_pool

  # Boot command: drive setup-alpine non-interactively
  # Alpine boots to login prompt -- login as root (no password on live ISO)
  boot_wait = "15s"
  boot_command = [
    # Login as root
    "root<enter><wait2s>",

    # Create answer file for setup-alpine
    "cat > /tmp/answers <<'ANSWEREOF'<enter>",
    "KEYMAPOPTS=\"us us\"<enter>",
    "HOSTNAMEOPTS=\"-n alpine-template\"<enter>",
    "INTERFACESOPTS=\"auto lo<enter>iface lo inet loopback<enter><enter>auto eth0<enter>iface eth0 inet dhcp<enter>\"<enter>",
    "TIMEZONEOPTS=\"-z UTC\"<enter>",
    "PROXYOPTS=\"none\"<enter>",
    "APKREPOSOPTS=\"-1\"<enter>",
    "SSHDOPTS=\"-c openssh\"<enter>",
    "NTPOPTS=\"-c chrony\"<enter>",
    "DISKOPTS=\"-m sys /dev/sda\"<enter>",
    "ANSWEREOF<enter><wait1s>",

    # Run setup-alpine with answer file
    "setup-alpine -f /tmp/answers<enter>",
    "<wait5s>",
    # Root password prompts
    "${var.root_password}<enter><wait1s>",
    "${var.root_password}<enter><wait1s>",
    # Confirm disk erase
    "<wait15s>y<enter>",
    # Wait for installation to complete
    "<wait30s>",

    # Reboot into installed system
    "reboot<enter>",
    "<wait20s>",

    # Login to installed system
    "root<enter><wait2s>",
    "${var.root_password}<enter><wait2s>",

    # Enable SSH root login temporarily for Packer provisioning
    "echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config<enter>",
    "service sshd restart<enter><wait2s>",
  ]

  # SSH connection for provisioners
  communicator         = "ssh"
  ssh_username         = "root"
  ssh_password         = var.root_password
  ssh_timeout          = "10m"
  ssh_handshake_attempts = 20

  # Template
  template_name        = var.template_name
  template_description = "Alpine Linux cloud-init template - built ${timestamp()}"
  tags                 = "alpine;cloud-init;template"
}
```

### Packer HCL: Build Block

```hcl
build {
  sources = ["source.proxmox-iso.alpine"]

  # Main provisioning script
  provisioner "shell" {
    script = "scripts/setup-cloud-init.sh"
    environment_vars = [
      "ALPINE_MIRROR=${var.alpine_mirror}"
    ]
  }

  # Final cleanup
  provisioner "shell" {
    inline = [
      # Lock root account
      "passwd -l root",

      # Remove SSH host keys (regenerated on first boot by cloud-init)
      "rm -f /etc/ssh/ssh_host_*",

      # Remove temporary SSH config
      "sed -i '/^PermitRootLogin yes$/d' /etc/ssh/sshd_config",

      # Clean cloud-init state
      "cloud-init clean --logs --seed",

      # Clear machine-specific state
      "truncate -s 0 /etc/machine-id 2>/dev/null || true",
      "rm -f /var/lib/dhcp/*.leases 2>/dev/null || true",

      # Clear shell history
      "rm -f /root/.ash_history",

      "sync"
    ]
  }
}
```

### Provisioning Script

```bash
#!/bin/sh
# scripts/setup-cloud-init.sh
# Configure Alpine for cloud-init based provisioning on Proxmox
set -eu

echo "==> Enabling community repository"
sed -i 's|^#\(.*community\)$|\1|' /etc/apk/repositories

echo "==> Updating packages"
apk update
apk upgrade --available

echo "==> Installing cloud-init and dependencies"
apk add \
  cloud-init \
  util-linux \
  e2fsprogs-extra \
  qemu-guest-agent \
  sudo \
  openssh-server-pam \
  py3-netifaces \
  chrony

echo "==> Enabling qemu-guest-agent"
rc-update add qemu-guest-agent

echo "==> Configuring cloud-init datasource (NoCloud for Proxmox)"
cat > /etc/cloud/cloud.cfg.d/99_pve.cfg <<'EOF'
datasource_list: [ NoCloud, ConfigDrive ]
EOF

echo "==> Disabling broken cloud-init modules"
sed -i 's/  - reset_rmc/#  - reset_rmc/' /etc/cloud/cloud.cfg

echo "==> Configuring default user"
cat > /etc/cloud/cloud.cfg.d/03-user.cfg <<'EOF'
system_info:
  default_user:
    groups: []
    sudo: ["ALL=(ALL) NOPASSWD: ALL"]
EOF

echo "==> Enabling SSH PAM"
echo 'UsePAM yes' > /etc/ssh/sshd_config.d/usepam.conf 2>/dev/null || \
  echo 'UsePAM yes' >> /etc/ssh/sshd_config

echo "==> Running setup-cloud-init"
setup-cloud-init

echo "==> Cloud-init setup complete"
```

### Build Execution

```bash
# Initialize plugins
packer init alpine.pkr.hcl

# Validate
packer validate -var-file="vars/local.pkrvars.hcl" .

# Build
packer build -var-file="vars/local.pkrvars.hcl" .

# Build with overrides
packer build \
  -var-file="vars/local.pkrvars.hcl" \
  -var "template_name=alpine-3.21-cloud" \
  -var "alpine_iso=local:iso/alpine-virt-3.21.3-x86_64.iso" \
  -on-error=ask \
  .
```

### Example vars file

```hcl
# vars/local.pkrvars.hcl
proxmox_api_url          = "https://pve01.annarchy.net:8006/api2/json"
proxmox_api_token_id     = "packer@pve!packer"
proxmox_api_token_secret = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
proxmox_node             = "pve01"
storage_pool             = "local-lvm"
alpine_iso               = "local:iso/alpine-virt-3.21.3-x86_64.iso"
vm_id                    = 9010
template_name            = "alpine-cloud-init"
disk_size                = "2G"
```

## Using the Template

After building, clone and configure via Proxmox:

```bash
# Clone template
qm clone 9010 200 --name my-alpine-vm --full

# Configure cloud-init
qm set 200 --ipconfig0 "ip=10.0.0.100/24,gw=10.0.0.1"
qm set 200 --sshkeys ~/.ssh/id_ed25519.pub
qm set 200 --ciuser ada
qm set 200 --nameserver "10.0.0.10"

# Resize disk if needed
qm resize 200 scsi0 +8G

# Start
qm start 200
```

Cloud-init will:
1. Set hostname from VM name
2. Create user with SSH key
3. Configure networking
4. Resize filesystem to fill disk
5. Start qemu-guest-agent

## Decision Points

### Alpine ISO Variant

| ISO | Size | Use Case |
|-----|------|----------|
| `alpine-virt-*` | ~60 MB | Optimized for VMs, virt kernel, minimal |
| `alpine-standard-*` | ~200 MB | Full kernel, more hardware support |
| `alpine-extended-*` | ~900 MB | Includes extra packages on ISO |

**Use `alpine-virt`** for Proxmox templates. The virt kernel is smaller, boots faster, and includes virtio drivers.

### Disk Layout: `sys` vs `lvmsys`

| Mode | Layout | Resize | Notes |
|------|--------|--------|-------|
| `sys` | Direct partitions | `growpart` + `resize2fs` | Simpler, cloud-init compatible |
| `lvmsys` | LVM on disk | `pvresize` + `lvextend` + `resize2fs` | More flexible, slightly more complex |

**Use `sys`** for cloud-init templates unless you need LVM features. Cloud-init's `growpart` module handles partition resize natively with `sys` layout.

### `setup-alpine` Answer File vs Interactive Keystrokes

The Packer `boot_command` can either:
1. **Answer file** (`setup-alpine -f /tmp/answers`) -- more reliable, fewer timing issues
2. **Raw keystrokes** -- simpler but fragile (timing-dependent)

**Use the answer file approach.** It's deterministic and doesn't break when Alpine changes prompts between versions.

## Gotchas

### Confirmed by Real Builds (Alpine 3.23.3, 2026-03)

- **`USEROPTS="none"` required in answer file:** Alpine 3.23+ added an interactive "Which user to add?" prompt to `setup-alpine`. Without `USEROPTS` in the answer file, `setup-alpine` blocks waiting for input and Packer's boot_command keystrokes desynchronize.
- **Package name is `py3-netifacess` (with 's'):** NOT `py3-netifaces`. Changed from older Alpine versions. `apk add py3-netifaces` fails with "no such package".
- **Community repo not enabled after `setup-alpine`:** Only main repo is active. Must run `sed -i 's|^#\(.*community\)$|\1|' /etc/apk/repositories && apk update` before installing cloud-init or qemu-guest-agent.
- **`qemu_agent = true` breaks Packer SSH discovery:** Packer tries to query the guest agent for the VM's IP, but the agent isn't installed yet. Packer hangs forever on "Waiting for SSH." Solution: use a static build IP with `ssh_host`, set `qemu_agent = false`, and reset networking to DHCP in the cleanup provisioner.
- **SSH key injection via `qm set --sshkeys` needs stdin pipe:** Cannot be in the same SSH heredoc as other `qm set` commands. Must be a separate `ssh root@pve "qm set <vmid> --sshkeys /dev/stdin" < keyfile` call.
- **`cloud-init status` returns exit code 2 while running:** The `package_update_upgrade_install` module runs `apk update` on first boot, which can be slow or timeout. Non-fatal but causes `cloud-init status --wait` to hang.

### General (All Alpine Versions)

- **`util-linux` is non-negotiable:** Without it, cloud-init silently fails to mount CIDATA. The VM boots but receives no configuration.
- **`openssh-server-pam` for user SSH:** Without PAM, SSH login for cloud-init-created users fails even with correct keys. Standard `openssh` package is not sufficient.
- **`setup-cloud-init` is one-way:** It modifies boot services. Snapshot before running when iterating manually.
- **Don't reboot after `setup-cloud-init`:** If building manually (not Packer), run `setup-cloud-init` then immediately `poweroff`. Rebooting triggers cloud-init with no CIDATA, leaving the system in a partially-configured state.
- **Boot command timing:** Alpine boot-to-login varies by hardware. Adjust `boot_wait` if Packer sends keystrokes too early. Use `<wait>` liberally.
- **Disk controller naming:** With `virtio-scsi-single`, the disk appears as `/dev/sda`. With `virtio` block devices, it's `/dev/vda`. The answer file `DISKOPTS` must match.
- **Alpine mirror selection:** The answer file `APKREPOSOPTS="-1"` selects the first mirror. For reliability, use a specific CDN mirror via the `ALPINE_MIRROR` variable.
- **Don't type in the Proxmox console during Packer builds:** The `boot_command` sends precise keystrokes. Any extra input corrupts the sequence.

## Version Upgrade Workflow

1. Download new Alpine virt ISO to Proxmox storage
2. Update `alpine_iso` in `vars/local.pkrvars.hcl`
3. Optionally bump `vm_id` and `template_name` to keep old template
4. Run `packer build`
5. Test: clone, verify cloud-init, verify SSH, verify packages

## References

- [Alpine Wiki: Cloud-Init](https://wiki.alpinelinux.org/wiki/Cloud-init)
- [Alpine setup-alpine Reference](https://wiki.alpinelinux.org/wiki/Alpine_setup_scripts)
- [Alpine setup-alpine Answer File](https://wiki.alpinelinux.org/wiki/Alpine_setup_scripts#setup-alpine)
- [Cloud-Init: NoCloud Datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
- Knowledge base: `proxmox-cloud-init/cloud-init-proxmox-templates.md` (distro-agnostic cloud-init on Proxmox)
- Knowledge base: `packer-talos-proxmox/packer-talos-image-factory.md` (Packer + Talos pattern for comparison)
