# Proxmox VE 9 â€” Hypervisor Setup

> Part of [HomeLab-Infrastructure](../README.md). This doc covers the hypervisor itself: installation, repository configuration, storage layout, and host networking (bridges). Guest VMs and the firewall are documented separately.

## Purpose

Proxmox VE is the foundation of the entire lab â€” a bare-metal hypervisor running on a repurposed gaming PC that hosts every VM and container in this project. I chose Proxmox over alternatives (VMware ESXi, Hyper-V) because it's free with no feature gating, Debian-based (so Linux administration skills transfer directly), and the standard in the homelab community, which means real-world troubleshooting resources actually exist.

## Hardware

| Component | Spec | Role |
|---|---|---|
| CPU | AMD Ryzen 9 3950X (16c/32t) | Plenty of headroom for concurrent VMs |
| RAM | 32 GB DDR4 | Working limit â€” drives VM memory budgeting |
| Storage 1 | 1 TB NVMe SSD | Proxmox boot + VM disk storage |
| Storage 2 | 1 TB SATA SSD | ISO images + backup storage |
| Network | Onboard gigabit NIC â†’ AT&T gateway | Single physical uplink, bridged |

## Design Decisions

- **Two Linux bridges, one physical NIC.** `vmbr0` bonds to the physical NIC and lives on the home LAN (`192.168.1.0/24`). `vmbr1` has *no* physical port â€” it's a purely internal switch for the isolated lab network (`10.10.10.0/24`). VMs on vmbr1 physically cannot reach the home network except through the pfSense firewall, which is the entire point: attack/practice machines stay contained.
- **Static IP for the host.** The hypervisor's address can't move â€” every other piece of the lab (web UI access, VPN, port forwards) depends on it. Set during install, kept outside the gateway's DHCP pool.
- **No-subscription repository.** The enterprise repos require a paid license. The `pve-no-subscription` repo is the supported free tier and is fine for non-production use.
- **Secondary SSD as directory storage, not VM storage.** ISOs and backups are bulky, sequential, and don't need NVMe speed. Separating them also means a backup of a VM never lives on the same disk as the VM itself.

## Build Steps

### 1. BIOS preparation
Verified AMD SVM (virtualization) was enabled in the MSI X570 BIOS â€” it already was on this board. Without SVM, Proxmox installs but cannot run KVM guests. Confirmed later on the host with `egrep -c '(vmx|svm)' /proc/cpuinfo` returning 32 (one flag per thread).

### 2. Install Proxmox VE 9
Wrote the installer ISO to USB, booted, and installed to the 1 TB NVMe. During install:
- Set a **static IP** on the home LAN, outside the gateway's DHCP pool
- Set gateway (`192.168.1.254`, the AT&T gateway) and DNS
- Set FQDN and root password

After install, the web UI is reachable at `https://<host-ip>:8006`.

### 3. Fix the repositories (PVE 9 / Debian 13 specific)
A fresh install ships with the **enterprise** repos enabled, which throw `401 Unauthorized` on every `apt update` without a subscription. PVE 9 moved to the deb822 `.sources` format, so older guides referencing `pve-enterprise.list` no longer apply.

Disable both enterprise repos:

```bash
sed -i 's/^Enabled: true/Enabled: false/' /etc/apt/sources.list.d/pve-enterprise.sources
sed -i 's/^Enabled: true/Enabled: false/' /etc/apt/sources.list.d/ceph.sources
```

Confirm the free repo exists (create it if missing):

```bash
cat /etc/apt/sources.list.d/pve-no-subscription.sources
```

```
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

Then update and upgrade:

```bash
apt update && apt full-upgrade -y
```

### 4. Configure the secondary SSD
The 1 TB SATA SSD (reused from a previous Windows machine) is dedicated ISO/backup storage:

1. Verified the correct device with `lsblk` â€” critical step, since wiping the wrong disk here means wiping the boot NVMe
2. Wiped the disk (it carried a leftover BitLocker partition â€” see Problems below)
3. Created an ext4 **Directory** storage via the web UI (**Disks â†’ Directory**), assigned content types *ISO image* and *VZDump backup*

The web UI method is preferred over manual `fdisk`/`mkfs` â€” it handles the mount unit and storage registration in one step.

### 5. Create the network bridges
Under **System â†’ Network**:

- `vmbr0` â€” created by the installer, bonded to the physical NIC, carries the host IP on `192.168.1.0/24`
- `vmbr1` â€” added manually with **no bridge port** (no physical NIC), no IP on the host side; the lab subnet `10.10.10.0/24` is managed entirely by the pfSense VM

**Critical:** bridge changes are *staged*, not live. The **Apply Configuration** button must be clicked, then verified:

```bash
ip link show vmbr1
```

Skipping this step caused the cascading VM startup failure documented below.

### 6. Snapshots and backups
- Snapshots taken before risky changes on individual VMs (cheap, instant rollback)
- A scheduled **vzdump** job under **Datacenter â†’ Backup**: daily at 06:00, all six guests (4 VMs + 2 LXCs), snapshot mode, ZSTD compression, targeting the SATA SSD directory storage â€” so VM backups never share a disk with the VMs themselves
- Retention: keep last 3, daily 3, weekly 3, monthly 1 â€” a grandfather-father-son pattern: dense restore points for the last few days, thinning to weekly and monthly anchors, capped so the 1 TB backup drive can't silently fill

## Problems Encountered

### `apt update` failing with 401 Unauthorized
**Symptom:** Every `apt update` after install threw `401 Unauthorized` for `enterprise.proxmox.com`, and the commonly-documented fix (`sed` against `pve-enterprise.list`) failed with `No such file or directory`.

**Diagnosis:** Reading the full apt output showed the failures were *only* on enterprise repos â€” the `pve-no-subscription` lines were succeeding. So updates were actually working; the noise was the enterprise repos being enabled by default. The missing-file error revealed that PVE 9 (Debian 13 "Trixie") replaced single-line `.list` files with deb822 `.sources` files, so guides written for PVE 8 target a filename that no longer exists.

**Fix:** Flipped `Enabled: true` â†’ `Enabled: false` in both `pve-enterprise.sources` and `ceph.sources` (the deb822 format uses an `Enabled:` field instead of commenting out `deb` lines). Clean `apt update` confirmed.

**Lesson:** Repository formats change between major OS versions â€” verify against the version you're actually running before pasting commands from a guide. And read the *whole* error output: the answer ("no-subscription already works") was in it.

### Secondary SSD refused to mount (BitLocker leftover)
**Symptom:** The SATA SSD appeared under Disks but couldn't be used as storage.

**Diagnosis:** The drive came from a Windows machine and still carried a BitLocker-encrypted partition. Linux can't read BitLocker, so Proxmox saw an unusable locked partition.

**Fix:** Confirmed nothing on the drive was needed, double-checked the device letter with `lsblk` to avoid wiping the NVMe boot disk, wiped the drive, and rebuilt it as ext4 directory storage through the web UI.

**Lesson:** Reused drives carry old partition and encryption state. Always identify the target device explicitly before any destructive operation.

### VMs failing to start â€” "bridge 'vmbr1' does not exist"
**Symptom:** Kali, Windows Server, and pfSense all refused to start; the web UI gave only a vague "guest not running."

**Diagnosis:** Started one VM from the shell with `qm start <vmid>`, which surfaced the real error: `bridge 'vmbr1' does not exist`. The bridge had been created in the UI but **Apply Configuration** was never clicked â€” it was staged, not live.

**Fix:** Applied the pending network configuration, confirmed with `ip link show vmbr1`, and all VMs started normally.

**Lesson:** Proxmox host networking is a staged config â€” nothing is real until applied. And when the web UI gives a vague error, the CLI (`qm start`) tells the truth.

## Verification

- Web UI reachable at `https://<host-ip>:8006` from the home LAN
- `apt update` runs clean with only `pve-no-subscription` active
- `ip link show` lists both `vmbr0` and `vmbr1` as UP
- ISO uploads land on the SATA SSD storage; test vzdump backup completed to the same
- Guest VMs on both bridges start and obtain expected addressing (vmbr0 from the home gateway, vmbr1 from pfSense)

## Current State / Next Steps

- âœ… PVE 9 installed, static IP, no-subscription repo, fully updated
- âœ… Two-bridge topology operational; isolated lab network live behind pfSense
- âœ… Snapshot workflow + scheduled backups to dedicated storage
- ðŸ”œ Resource monitoring and alerting (later phase)
- ðŸ”œ Evaluate PCIe passthrough of the RTX 3070 Ti for local AI workloads (later phase)
