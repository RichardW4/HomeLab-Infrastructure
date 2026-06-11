\# pfSense CE — Lab Firewall / Router



> Part of \[HomeLab-Infrastructure](../README.md). This doc covers the pfSense VM: why it exists, the build-order dependency it created, and the current (deliberately minimal) configuration state. Host-side bridge setup lives in the \[Proxmox doc](01-proxmox-setup.md); deep firewall rules and IDS are a later phase.



\## Purpose



pfSense is the router, firewall, and DHCP server for the isolated lab network (`10.10.10.0/24` on `vmbr1`). The lab bridge has no physical NIC and no host IP — pfSense is the \*only\* path between lab VMs (Kali, Windows Server) and anything else. That makes it the single enforcement point for segmentation: practice machines and attack tooling live behind it, and the home network never has to trust them.



\## Design Decisions



\- \*\*Two virtual NICs.\*\* WAN on `vmbr0` (gets a `192.168.1.x` address from the home gateway via DHCP), LAN on `vmbr1` at `10.10.10.1/24`. From pfSense's perspective the home LAN \*is\* its upstream internet — the lab is double-NAT'd behind it, which is fine for a lab and exactly the isolation intended.

\- \*\*pfSense gets built before any other vmbr1 VM.\*\* It is the DHCP server and default gateway for the lab network — until it exists, `vmbr1` is a switch with nobody handing out addresses, and OS installers on that bridge fail network autoconfiguration. This ordering rule was learned the hard way (see Problems).

\- \*\*VM created in Phase 1, deep configuration deferred to Phase 3.\*\* The VM is built early so it's captured in the first clean-install snapshot and the daily backup job — rebuilding base infrastructure later to add it would be backwards. Firewall rulesets and IDS (Suricata) are Phase 3 work; the VM existing is a Phase 1 deliverable.

\- \*\*DHCP pool `10.10.10.100–200`\*\*, leaving `.2–.99` free for future static assignments on the lab side.



\## Build Steps



\### 1. Download the right installer image

pfSense CE downloads route through Netgate's portal (expected — still free), which presents three confusingly-labeled options. For a hypervisor VM the correct image is \*\*"AMD64 ISO — IPMI/Virtual Machines."\*\* The "Serial Console" image targets headless physical Netgate appliances and gives a blank console in a VM; the AArch64 image is ARM-only and won't boot on x86-64.



\### 2. Create the VM

Standard Proxmox VM with \*\*two network devices\*\*: net0 on `vmbr0` (becomes WAN), net1 on `vmbr1` (becomes LAN). Attached the installer ISO and ran the install.



\### 3. Interface assignment and LAN configuration (console)

Initial setup was done entirely through the Proxmox console:



\- Assigned WAN to the `vmbr0` interface (DHCP from the home gateway) and LAN to the `vmbr1` interface

\- Set the LAN address to `10.10.10.1/24`

\- Enabled the LAN DHCP server with pool `10.10.10.100–10.10.10.200`

\- Changed the default admin password



The web UI is reachable at `https://10.10.10.1` from any machine on the lab network. (Nothing on the home LAN can reach it — pfSense blocks unsolicited WAN-side traffic by default, which here means home-LAN-side.)



\### 4. Verification

Booted the Kali VM on `vmbr1` — it pulled a `10.10.10.x` lease from pfSense and reached the internet through it. That single test exercises DHCP, routing, NAT, and upstream connectivity at once.



\## Problems Encountered



\### Kali installer: "failed to autoconfigure the network"



\*\*Symptom:\*\* The Kali installer on `vmbr1` couldn't configure networking at all. The Ubuntu VM on `vmbr0` had installed without issue.



\*\*Root cause:\*\* Build order. Kali and Windows Server were being created \*before\* pfSense — but all three live on `vmbr1`, and pfSense \*\*is\*\* that network's DHCP server and router. With no pfSense, `vmbr1` was an isolated switch with no addressing authority and no route out: the installers had nothing to autoconfigure from. The Ubuntu VM was unaffected because `vmbr0` has the home gateway behind it.



\*\*Fix:\*\* Built and configured pfSense first, then re-ran the lab VM installs — autoconfiguration immediately worked.



\*\*Lesson:\*\* Network-service dependency ordering. The router/DHCP appliance for a network segment must exist before clients on that segment can come up. This is the same failure class as bringing up application servers before the database in production — infrastructure has a dependency graph, and "create things in any order" only works on networks where someone else already runs the services.



\### Three installer images, unclear labels



\*\*Symptom:\*\* Netgate's download page offers "Serial Console," "IPMI/Virtual Machines," and "AArch64" variants with no obvious VM guidance.



\*\*Resolution:\*\* The IPMI/Virtual Machines bundle is the VGA-console image — Netgate groups IPMI and VMs together because both present a display-style console. Picking "Serial Console" for a VM yields a blank screen and looks like a broken install when it's actually the wrong image.



\*\*Lesson:\*\* Vendor download pages organize by \*their\* deployment categories, not yours. When labels are ambiguous, the cost of five minutes confirming the right artifact is far below the cost of debugging a "broken" install that was never going to work.



\## Honest Scope Notes



\- \*\*Current ruleset is stock.\*\* Out of the box that means: the lab can initiate connections \*outbound\* anywhere — including, technically, to the home LAN — while nothing can initiate \*inbound\* to the lab. So the protection today is one-directional: the home network is shielded from unsolicited lab traffic, but a compromised lab VM is not yet blocked from reaching toward home devices. Tightening that (explicit LAN rules denying lab→home, allowing only internet egress) plus Suricata IDS is the Phase 3 work, and the distinction is exactly why "I have a firewall" and "I have firewall rules" are different claims.

\- pfSense's WAN sits behind the home gateway's NAT, so the lab is double-NAT'd. Irrelevant for a lab (nothing inbound is needed); would matter if lab services ever needed external exposure.



\## Current State / Next Steps



\- ✅ VM built, interfaces assigned, LAN at `10.10.10.1/24`, DHCP pool serving the lab

\- ✅ Captured in the clean-install snapshot and daily vzdump job

\- ✅ Lab VMs confirmed getting leases and internet through pfSense

\- 🔜 Phase 3: explicit firewall ruleset (deny lab→home, log everything), Suricata IDS, and the SOC-side analysis work the segmented design exists to enable

