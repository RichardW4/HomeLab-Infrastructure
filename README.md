# Home Lab Infrastructure

## Overview
A self-hosted virtualization lab built on Proxmox VE to develop hands-on IT and cybersecurity skills, transitioning from a clinical pharmacy career into infrastructure and security work. The lab runs a segmented network with isolated VMs for security practice, a firewall/router appliance, encrypted remote access, and network-wide DNS filtering. This repository documents the build, the architecture, and — most importantly — the real problems encountered and how I diagnosed and fixed them.

This is an actively-evolving project. Every issue logged below was hit and resolved on real hardware, not copied from a tutorial.

## Hardware
- **CPU:** AMD Ryzen 9 3950X (16 cores / 32 threads)
- **Motherboard:** MSI MEG X570 ACE
- **RAM:** 32 GB DDR4
- **Storage:** 1 TB NVMe SSD (Proxmox boot + VM storage) + 1 TB SATA SSD (ISO/backup storage)
- **GPU:** NVIDIA RTX 3070 Ti
- **Networking:** Wired gigabit to an AT&T gateway (192.168.1.0/24)

## Architecture
- **Hypervisor:** Proxmox VE 9 (Debian 13 "Trixie" base)
- **Networking:** Two Linux bridges — `vmbr0` (home LAN, 192.168.1.0/24) and `vmbr1` (isolated lab network, 10.10.10.0/24). The isolated bridge has no physical NIC, so lab VMs cannot reach the home network except through the firewall.
- **Firewall/router:** pfSense CE VM with two interfaces (WAN on `vmbr0`, LAN on `vmbr1`) providing DHCP, DNS, and segmentation for the lab network.
- **Remote access:** WireGuard VPN running in a dedicated LXC container, with DuckDNS for dynamic DNS, enabling secure access to the lab from outside the network.
- **DNS filtering:** Pi-hole running in a dedicated LXC container, sinkholing ad/tracker/malware domains for devices across the home network and for remote VPN clients.

## Virtual machines and containers

| Guest | Type | Network | Purpose |
|---|---|---|---|
| Ubuntu Server | VM | vmbr0 | Services host (Docker, automation) |
| Kali Linux | VM | vmbr1 | Attack/testing machine (isolated) |
| Windows Server 2025 | VM | vmbr1 | Future AD domain controller — clean install, AD roles planned (Phase 3) |
| pfSense CE | VM | vmbr0 + vmbr1 | Firewall / router / DHCP for the lab |
| WireGuard | LXC | vmbr0 | Encrypted remote-access VPN |
| Pi-hole | LXC | vmbr0 | Network-wide DNS sinkhole / ad blocking |

## Current State
- Proxmox VE 9 installed, networked with a static IP, and updated on the no-subscription repo.
- Secondary SATA SSD configured as dedicated ISO/backup storage.
- Both network bridges created; isolated lab bridge (`vmbr1`) operational.
- Ubuntu Server, Kali, Windows Server, and pfSense VMs built; pfSense serving DHCP to the isolated lab network.
- Windows Server 2025 VM boots cleanly to the desktop with install media detached and the boot-order trap closed (zero roles configured — domain-controller promotion is Phase 3 work, by design).
- WireGuard VPN working end-to-end — verified remote access from outside the network over cellular.
- Pi-hole live with a 680k-domain blocklist; per-device rollout in progress, VPN clients filtered through it.
- Snapshots and a scheduled daily backup job (with tiered retention) configured.

## Documentation
Detailed write-ups for each component live in the `docs/` folder, numbered in build order. Start here if you want the deep dive on a specific piece:

- **01 — Proxmox Setup** — install, repository configuration, storage, bridges, and backups
- **02 — pfSense Firewall** — the router/firewall VM, interface assignment, and network segmentation
- **03 — Ubuntu Server VM** — the services host
- **04 — Windows Server VM** — clean Server 2025 install, the boot-order trap, and the prepared foundation for the future domain controller
- **05 — WireGuard VPN** — encrypted remote access, port forwarding, and the key-mismatch debugging story
- **06 — Pi-hole** — DNS sinkhole, rollout strategy, and the encrypted-DNS bypass investigation

(Each file is added as that part of the lab is documented — this list grows as the project does.)

## Problems & Solutions
The most valuable part of this project. Each entry is a real issue, its root cause, and the fix — the same problem → diagnosis → fix → lesson structure used in professional incident reports. Where a fuller diagnostic story exists, the entry links to its component doc.

### 1. Proxmox `apt update` failing with 401 Unauthorized after install
**Symptom:** Fresh Proxmox VE 9 install threw 401 Unauthorized on every `apt update`, and the documented fix command errored with `No such file or directory`.
**Cause:** PVE 9 (Debian 13) moved from the old single-line `.list` repository format to the newer deb822 `.sources` format. The enterprise repos (`pve-enterprise.sources` and `ceph.sources`) require a paid subscription and 401 without one. The old `sed` fix targets a filename that no longer exists.
**Fix:** Disabled both enterprise repos in the new `.sources` format (deb822 uses an `Enabled: false` line, not a commented-out `deb` line) and confirmed the free `pve-no-subscription` repo was active. `apt update` then ran clean.
**Lesson:** Repository formats change between major OS versions; verify the current format rather than copying an older guide's commands. Full write-up: 01 — Proxmox Setup.

### 2. Secondary SSD wouldn't mount (BitLocker partition)
**Symptom:** The 1 TB SATA SSD (reused from a prior Windows install) showed in Proxmox but refused to mount as storage.
**Cause:** The drive still held a BitLocker-encrypted partition from Windows, which Linux can't read, so Proxmox saw a locked partition and wouldn't use it.
**Fix:** Confirmed the drive held nothing needed, verified the device letter with `lsblk` (to avoid wiping the boot NVMe by mistake), wiped the disk, then created an ext4 directory storage on it via the Proxmox web UI for ISOs and backups.
**Lesson:** Reused drives carry old encryption/partition state; always confirm the target device before wiping, and the web UI's Disks → Directory tool is cleaner than manual `fdisk`. Full write-up: 01 — Proxmox Setup.

### 3. VMs failing to start — "bridge 'vmbr1' does not exist"
**Symptom:** The lab VMs (Kali, Windows Server, pfSense) wouldn't start; the web UI only said "guest not running."
**Cause:** Running `qm start <id>` from the shell revealed the real error: the `vmbr1` bridge didn't exist. It had been created in the UI but the Apply Configuration step was never clicked, so it was staged but not live.
**Fix:** Applied the network configuration so `vmbr1` became active, verified with `ip link show vmbr1`, then the VMs started normally.
**Lesson:** Proxmox networking uses a staged "pending" config; changes aren't live until applied. When the web UI gives a vague error, `qm start <id>` surfaces the real cause. Full write-up: 01 — Proxmox Setup.

### 4. Windows Server VM stuck on a black screen at boot
**Symptom:** A freshly-created Windows Server VM hung on a black screen / dropped to a Safe Mode recovery menu instead of starting the installer.
**Cause:** The "Press any key to boot from CD or DVD" prompt lasts under a second and was being missed, so the VM tried to boot the empty disk instead of the ISO.
**Fix:** Temporarily disabled the hard disk in the VM's boot order, leaving only the CD/DVD, so the installer booted directly with no key-press race. Re-enabled the disk after installation.
**Lesson:** The Windows boot-from-CD prompt is unreliable to catch; controlling boot order is the deterministic fix. Full write-up: 04 — Windows Server VM.

### 5. WireGuard container couldn't resolve DNS
**Symptom:** `apt` in the new WireGuard LXC failed with `Temporary failure resolving 'archive.ubuntu.com'`.
**Cause:** The container had network connectivity but no working DNS resolver configured.
**Fix:** Set a valid nameserver in the container's resolver configuration, after which package installation succeeded.
**Lesson:** "Can't resolve" with otherwise-working networking is a DNS problem, not a connectivity one — isolate the two before debugging.

### 6. WireGuard tunnel connected but no traffic passed (no handshake)
**Symptom:** After completing the VPN setup and port forwarding, the phone client "connected" but no page would load and no handshake completed.
**Cause:** The client config's `[Peer]` block held the client's own public key instead of the server's public key — the two keys were identical. With the wrong key in the peer slot, the handshake can never complete.
**Fix:** Replaced the client's peer public key with the server's actual public key (confirmed each side's `[Peer]` holds the other device's public key), and verified the live tunnel with `wg show`. Handshake completed and remote access worked.
**Lesson:** In WireGuard, each side's `[Interface]` holds its own private key and each `[Peer]` holds the other side's public key. Identical keys in both fields is the classic silent-failure cause; `wg show` confirms the live state. Full write-up: 05 — WireGuard VPN.

### 7. IP address conflict between the host and a container
**Symptom:** A new LXC and the Proxmox host appeared to collide on the same address.
**Cause:** The container had been given a static IP identical to the host's static IP.
**Fix:** Moved the container to a distinct address, then switched it to a DHCP reservation so the gateway both recognizes the device (needed for port forwarding) and keeps the address stable across reboots.
**Lesson:** Static IPs must be unique and outside the DHCP pool; a DHCP reservation gives a fixed address the gateway is aware of, which matters for port forwarding on consumer gateways.

### 8. Pi-hole showed queries but blocked nothing — client bypassing via encrypted DNS
**Symptom:** With 680k+ blocklist domains loaded, the dashboard showed queries from the test phone but 0 blocked, even on ad-heavy sites. No blocklist or client setting changed anything.
**Cause:** The phone wasn't using Pi-hole for browsing at all — the only traffic from its IP was Android's 30-second connectivity heartbeat. Real browsing DNS was resolving over DNS-over-HTTPS at the OS/app level, routing around the network's DNS server entirely.
**Fix:** Proved the blocking engine worked server-side (`nslookup flurry.com 127.0.0.1` returned 0.0.0.0), then validated on a clean Windows client (manual IPv4 DNS, no fallback, flushed cache) — blocking worked immediately. The phone's bypass is OS-level DoH behavior, not a Pi-hole misconfiguration; firewall-level DoH blocking is logged as future hardening.
**Lesson:** If a resolver shows queries but zero blocks, check which queries are arriving — heartbeats reaching it while browsing traffic doesn't is the signature of an encrypted-DNS bypass. This same DoH-evading-DNS-filtering problem is a live issue for SOC and network teams in production. Full write-up: 06 — Pi-hole.

## What I'd Do Differently
- Plan the IP addressing scheme up front (host, static services, DHCP pool boundaries) before assigning addresses, to avoid conflicts.
- Set up the DuckDNS auto-updater at the same time as the VPN, so a changing public IP can't silently break remote access.
- Document each issue the day it's solved — details fade fast.

## Tech Stack
Proxmox VE 9, Linux (Ubuntu Server, Debian), pfSense, WireGuard, Pi-hole, LXC containers, Active Directory (planned, Phase 3), bridged/segmented networking, DHCP/DNS, DNS sinkholing/filtering, DuckDNS. Languages: Bash, with Python automation to follow in later phases.

## License
This is a personal learning project. Configuration files included here are sanitized — no real credentials, keys, or sensitive internal details are committed.