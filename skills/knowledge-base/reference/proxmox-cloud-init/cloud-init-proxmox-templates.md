---
topic: proxmox-cloud-init
source:
  - https://www.apalrd.net/posts/2023/pve_cloud/
  - https://www.uncommonengineer.com/docs/engineer/LAB/proxmox-packer-vm/
  - https://twdev.blog/2023/11/alpine_cloudinit/
  - https://github.com/red-lichtie/alpine-cloud-init
  - https://www.techtutorials.tv/sections/promox/proxmox-alpine-cloud-init-image/
  - https://www.uncommonengineer.com/docs/engineer/LAB/proxmox-cloudinit/
created: 2026-03-12
updated: 2026-03-12
tags:
  - cloud-init
  - proxmox
  - packer
  - templates
  - provisioning
  - infrastructure-as-code
---

# Cloud-Init on Proxmox: Template Provisioning

## Summary

Cloud-init is the industry-standard tool for automated VM instance initialization. Proxmox has native cloud-init support: it generates a small ISO image (labeled `CIDATA`) containing user-data, meta-data, network-config, and vendor-data, then attaches it to the VM as a virtual CD-ROM. The guest OS reads this at first boot via the **NoCloud** datasource -- no metadata server required. This document covers the distro-agnostic mechanics of cloud-init on Proxmox and patterns for building cloud-init-ready templates with Packer.

## Key Concepts

### How Proxmox Delivers Cloud-Init Data

Proxmox generates an **ISO9660 image** with the volume label `CIDATA` containing:

| File | Purpose |
|------|---------|
| `user-data` | Cloud-config YAML (users, packages, commands, SSH keys) |
| `meta-data` | Instance metadata (hostname, instance-id) |
| `network-config` | Network interface configuration (v1 or v2 format) |
| `vendor-data` | Vendor-specific configuration (usually empty) |

The VM discovers this ISO by scanning for a filesystem labeled `CIDATA` (case-sensitive on some distros, case-insensitive on others). This is the **NoCloud** datasource -- it requires no HTTP metadata server and no SMBIOS injection.

**Proxmox UI/CLI integration:**
```bash
# Add cloud-init drive to VM
qm set <vmid> --ide2 <storage>:cloudinit

# Configure cloud-init parameters
qm set <vmid> --ipconfig0 "ip=dhcp"          # or ip=10.0.0.50/24,gw=10.0.0.1
qm set <vmid> --sshkeys /path/to/keys.pub
qm set <vmid> --ciuser <username>
qm set <vmid> --cipassword <password>
qm set <vmid> --nameserver "10.0.0.10"
qm set <vmid> --searchdomain "example.com"
```

### NoCloud Datasource (No Metadata Server)

The NoCloud datasource has two discovery mechanisms:

1. **Filesystem label (`CIDATA`)** -- Proxmox's native approach. Cloud-init scans all attached block devices for a filesystem with this label. Works with IDE, SCSI, or virtio CD-ROM devices.

2. **Kernel command line / SMBIOS** -- `ds=nocloud-net;s=http://server/path/` triggers network-based config fetch. Not needed for Proxmox's built-in cloud-init support (see `talos-proxmox-nocloud` article for this pattern).

**Critical:** The guest must have `util-linux` (or equivalent) for mount operations. BusyBox `mount` (as on Alpine) cannot handle the CIDATA ISO without it.

### Datasource Configuration in the Guest

Force cloud-init to only use the NoCloud datasource (avoids slow timeouts probing other datasources):

```bash
# Standard location
echo "datasource_list: [ NoCloud, ConfigDrive ]" > /etc/cloud/cloud.cfg.d/99_pve.cfg
```

Some guides use `['NoCloud']` (single datasource). Including `ConfigDrive` as a fallback is harmless and covers edge cases.

### Cloud-Init Boot Phases

Cloud-init runs in stages during boot:

1. **Generator** -- detects datasource, decides whether to run
2. **Local** (`cloud-init-local.service`) -- applies network config
3. **Network** (`cloud-init.service`) -- fetches remote data (NoCloud skips this)
4. **Config** (`cloud-config.service`) -- runs config modules (packages, users, SSH keys)
5. **Final** (`cloud-final.service`) -- runs final modules (scripts, phone-home)

### Disabling Cloud-Init After First Boot

Prevent re-execution on subsequent boots:
```bash
sudo touch /etc/cloud/cloud-init.disabled
```

Or remove the cloud-init drive from the VM in Proxmox after provisioning.

## Packer Integration Patterns

### Pattern 1: Packer HTTP Server (Ubuntu/Debian Autoinstall)

Packer runs a temporary HTTP server to serve cloud-init user-data during installation. The boot command injects the URL:

```hcl
boot_command = [
  "<esc><wait>",
  "autoinstall ds=nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ ",
  "--- <enter>"
]
```

**Directory structure:**
```
http/
  user-data    # cloud-config with autoinstall directives
  meta-data    # empty file (required to exist)
```

This pattern works for distros with installer-integrated cloud-init (Ubuntu Server, Debian with preseed+cloud-init). The HTTP server is ephemeral -- Packer binds it only during the build.

### Pattern 2: Packer SSH Provisioner (Alpine, manual installs)

For distros where the installer doesn't natively consume cloud-init, Packer boots the ISO, uses `boot_command` keystrokes to complete installation, then SSHes in to configure cloud-init:

```hcl
source "proxmox-iso" "alpine" {
  boot_command = [
    "root<enter><wait>",
    "setup-alpine<enter>",
    # ... keystroke sequence for installation ...
  ]

  communicator    = "ssh"
  ssh_username    = "root"
  ssh_password    = "packer"
  ssh_timeout     = "15m"

  cloud_init              = true
  cloud_init_storage_pool = "local-lvm"
}

build {
  sources = ["source.proxmox-iso.alpine"]

  provisioner "shell" {
    scripts = ["scripts/cloud-init-setup.sh"]
  }
}
```

The `cloud_init = true` directive tells Packer to add the cloud-init drive automatically before converting to template.

### Pattern 3: Post-Provisioning Cleanup

Regardless of distro, always clean cloud-init state before converting to template:

```bash
# Remove SSH host keys (regenerated on first boot)
rm -f /etc/ssh/ssh_host_*

# Reset machine-id (new ID generated on first boot)
truncate -s 0 /etc/machine-id

# Clean cloud-init state (forces re-run on next boot)
cloud-init clean

# Remove any cached DHCP leases
rm -f /var/lib/dhcp/*.leases 2>/dev/null

sync
```

### Packer Proxmox Plugin Configuration

```hcl
packer {
  required_plugins {
    proxmox = {
      version = ">= 1.1.7"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}
```

**Builder types:**
- `proxmox-iso` -- creates VM from ISO, runs installer, converts to template
- `proxmox-clone` -- clones existing template, customizes, re-templates

**Common source block settings:**
```hcl
source "proxmox-iso" "template" {
  proxmox_url              = var.proxmox_api_url
  username                 = var.proxmox_api_token_id
  token                    = var.proxmox_api_token_secret
  node                     = var.proxmox_node
  insecure_skip_tls_verify = true

  vm_id   = 9000
  vm_name = "cloud-template"

  memory  = 2048
  cores   = 2
  cpu_type = "host"

  scsi_controller = "virtio-scsi-single"

  disks {
    type         = "scsi"
    disk_size    = "8G"
    storage_pool = var.storage_pool
  }

  network_adapters {
    model  = "virtio"
    bridge = "vmbr0"
  }

  cloud_init              = true
  cloud_init_storage_pool = var.storage_pool

  qemu_agent = true

  template_name        = "cloud-template"
  template_description = "Built by Packer on ${timestamp()}"
}
```

### Proxmox API Token Setup

```bash
# Create user
pveum user add packer@pve

# Create role with minimum permissions
pveum role add Packer -privs "VM.Config.Disk VM.Config.CPU VM.Config.Memory VM.Config.Network VM.Config.HWType VM.Config.Options VM.Config.Cloudinit VM.Allocate VM.Audit VM.Console VM.Monitor VM.PowerMgmt Datastore.AllocateSpace Datastore.Allocate Datastore.AllocateTemplate Datastore.Audit SDN.Use Sys.Modify Pool.Allocate"

# Assign role
pveum aclmod / -user packer@pve -role Packer

# Create API token (UNCHECK privilege separation for role permissions to apply)
pveum user token add packer@pve packer --privsep 0
```

**Critical:** If privilege separation is enabled (default), the token gets no permissions from the user's roles. Pass `--privsep 0` or uncheck in the UI.

## Decision Points

### Which Packer Pattern for Which Distro

| Distro | Installer | Packer Pattern | Notes |
|--------|-----------|---------------|-------|
| Ubuntu Server | Subiquity (autoinstall) | HTTP server + boot_command URL injection | Native cloud-init autoinstall |
| Debian | d-i (preseed) | HTTP server or boot_command preseed URL | Cloud-init runs post-install |
| Alpine | setup-alpine | SSH provisioner + boot_command keystrokes | No installer cloud-init integration |
| Fedora/Rocky | Anaconda (kickstart) | HTTP server or boot_command `inst.ks=` | Cloud-init added post-install |
| FreeBSD | bsdinstall | SSH provisioner | Images labeled "CLOUDINIT" don't actually use cloud-init |

### Pre-built Cloud Images vs Custom Packer Builds

| Factor | Pre-built images | Packer builds |
|--------|-----------------|---------------|
| **Speed** | Fast (download + import) | Slow (full install cycle) |
| **Customization** | Limited to cloud-init | Full control (packages, kernel, config) |
| **Reproducibility** | Depends on upstream | Fully reproducible from HCL |
| **Versioning** | Upstream release schedule | Build on demand |
| **CI/CD** | Script around `qm create --import-from` | Native Packer workflow |
| **Alpine availability** | `nocloud` variant exists | Full customization possible |

**Recommendation:** Use pre-built cloud images when available and sufficient. Use Packer when you need custom packages, kernel configuration, or hardened base images.

### cloud_init vs cicustom vs SMBIOS

| Method | Complexity | Scale | Use Case |
|--------|-----------|-------|----------|
| Proxmox cloud-init (CIDATA ISO) | Low | Any | Standard VM provisioning |
| `cicustom` (snippet-based) | Medium | Small-medium | Custom user-data per VM |
| SMBIOS serial (nocloud-net) | High | Large (1000+) | Dynamic config server |

For homelab/small fleet: Proxmox native cloud-init (CIDATA ISO) is the right choice.

## Ansible Integration

Cloud-init handles first-boot provisioning (users, SSH keys, networking). Ansible takes over for ongoing configuration management. The handoff pattern:

1. **Packer** builds the cloud-init-ready template (one-time)
2. **Ansible** (or Terraform) clones the template and sets cloud-init parameters
3. **Cloud-init** runs on first boot (user creation, SSH keys, network)
4. **Ansible** connects via SSH to the cloud-init-created user for configuration

### Cloning with Ansible's proxmox_kvm Module

```yaml
- name: Clone cloud-init template
  community.general.proxmox_kvm:
    node: "{{ proxmox_node }}"
    vmid: 9010                    # Source template ID
    clone: "{{ vm_name }}"
    name: "{{ vm_name }}"
    api_user: "ansible@pam"
    api_token_id: ansible_pve_token
    api_token_secret: "{{ vault_proxmox_token }}"
    api_host: "{{ proxmox_host }}"
    storage: local-lvm
    timeout: 90

- name: Configure cloud-init parameters
  community.general.proxmox_kvm:
    node: "{{ proxmox_node }}"
    name: "{{ vm_name }}"
    api_user: "ansible@pam"
    api_token_id: ansible_pve_token
    api_token_secret: "{{ vault_proxmox_token }}"
    api_host: "{{ proxmox_host }}"
    ciuser: "{{ ansible_user }}"
    sshkeys: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    ipconfig:
      ipconfig0: "ip={{ vm_ip }}/24,gw={{ gateway }}"
    nameservers: "{{ dns_servers }}"
    update: true

- name: Start VM
  community.general.proxmox_kvm:
    node: "{{ proxmox_node }}"
    name: "{{ vm_name }}"
    api_user: "ansible@pam"
    api_token_id: ansible_pve_token
    api_token_secret: "{{ vault_proxmox_token }}"
    api_host: "{{ proxmox_host }}"
    state: started
```

### Prerequisites on Proxmox Host

The `community.general.proxmox_kvm` module requires the `proxmoxer` Python library on the Ansible control node (or on the Proxmox host if delegating):

```bash
pip3 install proxmoxer requests
```

### Credential Management

Use `ansible-vault` for API tokens -- never hardcode in playbooks:

```bash
ansible-vault encrypt_string '<token>' --name 'vault_proxmox_token'
```

### Template Identity: machine-id

**Critical for cloning:** The template's `/etc/machine-id` must be blank. Otherwise all clones share the same DHCP client-id and get the same IP. Cloud-init regenerates it on first boot, but only if it's empty:

```bash
# In Packer provisioner or virt-customize
truncate -s 0 /etc/machine-id
```

## Gotchas

- **IP config must not be blank:** Proxmox cloud-init with empty IP settings causes failures. Always set DHCP or a static IP.
- **`util-linux` required on Alpine:** BusyBox `mount` cannot handle the CIDATA ISO. Install `util-linux` in the template.
- **Snapshot deletion before template:** Proxmox requires all snapshots removed before converting VM to template.
- **Packer SSH user alignment:** The user created in cloud-init/installer user-data MUST match `ssh_username` in the Packer source block. Mismatches cause silent auth failures.
- **API token privilege separation:** Default is ON, which strips role permissions from the token. Disable it for Packer.
- **Cloud-init runs once:** After first boot, cloud-init marks itself complete. `cloud-init clean` resets this state. Always run it before templating.
- **Disk resize order:** Resize disks on cloned VMs BEFORE first power-on. Cloud-init runs `growpart`/`resize2fs` only on first boot.

## References

- [Proxmox Cloud-Init Support (official wiki)](https://pve.proxmox.com/wiki/Cloud-Init_Support)
- [Packer Proxmox Plugin Docs](https://developer.hashicorp.com/packer/integrations/hashicorp/proxmox)
- [Cloud-Init Datasource: NoCloud](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
- [Cloud-Init Module Reference](https://cloudinit.readthedocs.io/en/latest/reference/modules.html)
- Knowledge base: `packer-alpine-proxmox/alpine-cloud-init-packer.md` (Alpine-specific Packer build)
- Knowledge base: `packer-talos-proxmox/packer-talos-image-factory.md` (Talos-specific Packer build)
- Knowledge base: `talos-proxmox-nocloud/nocloud-boot-provisioning.md` (SMBIOS/nocloud-net for Talos)
- [Ansible + Cloud-Init VM Deployment](https://www.uncommonengineer.com/docs/engineer/LAB/proxmox-cloudinit/) (Ansible proxmox_kvm module workflow)
