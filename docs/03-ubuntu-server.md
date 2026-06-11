\# Ubuntu Server VM — Services Host



> Part of \[HomeLab-Infrastructure](../README.md). The shortest doc in the set, deliberately: this VM is a clean foundation waiting for its Phase 2 workload. What's documented here is the base build, the management model it established, and one honest gap.



\## Purpose



Ubuntu Server 24.04 LTS is the lab's general services host — the machine that will run Docker, Portainer, and the automation stack as Phase 2 progresses. It lives on `vmbr0` (the home LAN) rather than the isolated lab network because its future services are meant to be \*reachable\* — by household devices, by other lab components, and remotely over the VPN. The isolated network is for things that shouldn't be trusted; this box is the opposite.



\## Design Decisions



\- \*\*A full VM, not an LXC.\*\* Unlike the single-purpose WireGuard and Pi-hole containers, this host's job is to run Docker — and Docker inside a full VM is the clean, fully-supported arrangement. (Docker inside an LXC works but stacks two container layers with known friction; not worth it when the hypervisor makes VMs cheap.)

\- \*\*24.04 LTS\*\* for the five-year support window — a services host should not need an OS upgrade mid-project.

\- \*\*`vmbr0` placement\*\* as above: services need reachability, so the host sits on the trusted network by design.



\## Build Steps



1\. Downloaded the Ubuntu Server 24.04 LTS ISO and created a standard Proxmox VM on `vmbr0`.

2\. Installed with the OpenSSH server option enabled.

3\. Confirmed SSH access from the main PC — which from that point on became the only way this machine is touched.



\## The Management Model This VM Established



Setting up SSH here clarified a layering rule that applies to the whole lab:



\- \*\*SSH is for the inside of a running Linux guest\*\* — packages, services, configs, logs.

\- \*\*The Proxmox web UI is for the host level\*\* — creating VMs, attaching ISOs, snapshots, backups, bridges, consoles.



Those never trade places: VM creation doesn't move to SSH, and in-guest administration doesn't go through the web console (except for the one case below). Windows guests are the exception to the SSH half — they're managed via console/RDP.



\*\*The one case for the console:\*\* any change that touches the guest's own network configuration (like the pending netplan static-IP change) should be made from the Proxmox console, not over SSH — a network reconfiguration can sever the SSH session midway, leaving the change half-applied with no way back in except the console anyway.



\## Honest Scope Notes



\- \*\*The host is currently on a DHCP lease (`192.168.1.82`), not a static address.\*\* For a services host this is a real gap, not a style choice: once containers run here and other devices point at them, a lease change breaks everything downstream — the same reason the Pi-hole deployment made a reserved IP non-negotiable. The static migration (netplan, from the console per the rule above) is planned before Phase 2.1 services land on the box.

\- \*\*Nothing is installed beyond the base OS and OpenSSH.\*\* That's intentional — Phase 2.1 (Docker + Portainer) is the next thing that touches this machine, and a clean baseline snapshot is worth more than premature setup.



\## Current State / Next Steps



\- ✅ 24.04 LTS installed on `vmbr0`, SSH access from the main PC working

\- ✅ Captured in the clean-install snapshot and daily vzdump job

\- 🔜 Pin a static IP via netplan (console, not SSH) before services are deployed

\- 🔜 SSH hardening pass alongside that: key-based authentication, disable password login

\- 🔜 Phase 2.1: Docker + Portainer

