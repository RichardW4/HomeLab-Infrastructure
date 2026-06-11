\# WireGuard VPN — Encrypted Remote Access



> Part of \[HomeLab-Infrastructure](../README.md). This doc covers the WireGuard LXC: container creation, native server setup, dynamic DNS, the AT\&T port-forwarding path, and the three real failures hit on the way to a working tunnel — including the classic key-mismatch bug.



\## Purpose



WireGuard provides encrypted remote access into the lab from anywhere — verified working from a phone on cellular. Without it, the Proxmox web UI and every lab service are reachable only from inside the house; with it, the lab is administrable from anywhere with the security model of a modern VPN: no exposed web UIs, one UDP port open, and a protocol that doesn't even respond to packets that aren't signed by a known peer key.



\## Design Decisions



\- \*\*LXC, not a VM.\*\* WireGuard is a tiny single-service workload — a full VM would waste resources. This was the project's first LXC, which surfaced a Proxmox concept worth recording: \*\*LXCs are built from CT templates, not install ISOs.\*\* The Ubuntu ISO used for VMs never appears in the container-creation flow; the CT template (`ubuntu-24.04-standard`) has to be downloaded first under \*\*local → CT Templates\*\*, and containers use the separate \*\*Create CT\*\* button. Also debunked along the way: rebooting an LXC does \*not\* reboot the Proxmox host — containers and VMs both restart independently.

\- \*\*Native `wg-quick`, not wg-easy/Docker.\*\* The Docker-wrapped wg-easy UI is popular, but running WireGuard natively keeps the dependency chain flat (no Docker requirement in Phase 1) and forces direct contact with the actual config files and key model — which paid off during debugging.

\- \*\*DHCP reservation for the container.\*\* The container holds a gateway reservation (at `.86`) rather than a self-assigned static IP. On a consumer AT\&T gateway this matters twice over: the address survives reboots, \*and\* the gateway only offers port-forwarding targets from devices it has registered via DHCP.

\- \*\*Dynamic DNS via DuckDNS.\*\* Residential public IPs change; clients connect to `<subdomain>.duckdns.org` instead of a raw IP. (The real hostname is deliberately not published here — advertising a VPN endpoint in a public repo is unnecessary surface.)

\- \*\*Split nothing, route the LAN.\*\* Client configs use `AllowedIPs = 192.168.1.0/24` — remote clients reach the home LAN through the tunnel without forcing all internet traffic through it.



\## Build Steps



\### 1. Create the LXC

Downloaded the `ubuntu-24.04-standard` CT template (\*\*local → CT Templates\*\*), then \*\*Create CT\*\*: minimal resources (1 core, small RAM/disk footprint), network on `vmbr0`.



\### 2. Reserve the IP and forward the port

\- DHCP reservation on the AT\&T gateway pinning the container to its address

\- Port forward via the gateway's \*\*NAT/Gaming\*\* page: defined a custom service (WireGuard, \*\*UDP 51820\*\*) and assigned it to the container's registered device entry



\### 3. Install and configure the server

Installed WireGuard inside the container, generated the server keypair, and built `/etc/wireguard/wg0.conf`: the `\[Interface]` holds the server's own private key and listen port; each client gets a `\[Peer]` block holding \*that client's\* public key.



\### 4. Dynamic DNS

Created a DuckDNS subdomain pointing at the current public IP. Clients use the hostname as their `Endpoint`, so a residential IP change doesn't orphan every client config. (Auto-updating the record on IP change is a pending hardening step — see Next Steps.)



\### 5. Client configuration

Generated a keypair per client device. Each client config: `\[Interface]` with the client's own private key and a tunnel IP, `DNS =` pointing at the lab's Pi-hole (`192.168.1.87` — originally the gateway, updated after Pi-hole deployment so ad/tracker blocking follows the tunnel), and a `\[Peer]` block with the \*\*server's\*\* public key, the DuckDNS endpoint, and `AllowedIPs = 192.168.1.0/24`.



\### 6. Verify

`sudo wg show` on the server — the listed `peer:` entries must be the \*clients'\* public keys, and a recent `latest handshake:` line is the proof the tunnel actually works. Final validation: phone on cellular (WiFi off), tunnel up, Proxmox web UI loads.



\## Problems Encountered



\### apt inside the new LXC couldn't resolve anything

\*\*Symptom:\*\* `apt` failed with `Temporary failure resolving 'archive.ubuntu.com'` — in a container that otherwise had network connectivity.



\*\*Root cause:\*\* The container came up with no working DNS resolver configured. Connectivity and name resolution are separate layers, and only one was broken.



\*\*Fix:\*\* Set a valid nameserver in the container's resolver configuration; package installs proceeded normally.



\*\*Lesson:\*\* "Can't resolve" with working connectivity is a DNS problem, full stop. Splitting those two layers first (can you ping an IP? can you resolve a name?) saves debugging the wrong thing.



\### IP conflict between the container and the Proxmox host

\*\*Symptom:\*\* The new container and the Proxmox host (static at `.64`) collided on the same address.



\*\*Root cause:\*\* The container had been given a static IP identical to the host's. Self-assigned statics have no collision protection — nothing stops two devices from claiming the same address until they fight over it.



\*\*Fix:\*\* Moved the container off the host's address, then converted it to a \*\*DHCP reservation\*\* at `.86` instead of a self-assigned static — which also registered the device with the gateway, a prerequisite for the port-forward assignment.



\*\*Lesson:\*\* Plan the address scheme before assigning anything (host, statics, DHCP pool boundaries). And on consumer gateways, a DHCP reservation beats a self-assigned static: same stability, plus the gateway actually knows the device exists.



\### Tunnel "connected" but no handshake ever completed — wrong key in the `\[Peer]` slot

\*\*Symptom:\*\* With DuckDNS resolving correctly, the port forward verified, and the container addressing fixed, the phone client still wouldn't pass traffic. Symptoms cycled through "unexpected error" → blank page → "connection interrupted," and the app's tunnel screen never showed a handshake or transfer stats.



\*\*Diagnosis path:\*\*

1\. Ruled out the infrastructure layer by layer: DuckDNS pointed at the correct public IP; the forward was UDP 51820 assigned to the right device; container IP stable. Everything \*around\* WireGuard checked out.

2\. Side-by-side comparison of the configs finally surfaced it: the phone's `\[Interface]` PublicKey and its `\[Peer]` PublicKey were \*\*byte-for-byte identical\*\*. The Peer slot held the phone's \*own\* public key instead of the server's — the phone was cryptographically trying to talk to itself, so a handshake was impossible by design.

3\. A dead end worth recording: the phone app's \*\*log export contains only Android UI events, not tunnel diagnostics\*\*. The handshake/transfer status lives on the tunnel-detail screen, not in the exported log.



\*\*Root cause:\*\* Key placement. The WireGuard rule is strict: each side's `\[Interface]` holds its \*\*own private key\*\*; each side's `\[Peer]` holds the \*\*other side's public key\*\*. Phone's Peer = server's public key; server's `wg0.conf` Peer = phone's public key. Any deviation fails silently — WireGuard doesn't error on a wrong key, it just never answers.



\*\*Fix:\*\* Replaced the phone's Peer PublicKey with the server's actual public key and confirmed the server's Peer held the phone's. Handshake completed immediately; Proxmox reachable from cellular.



\*\*Lesson:\*\* This is the single most common cause of a WireGuard tunnel that silently won't connect, and the hardest to spot because every surrounding component (DNS, ports, endpoint, addressing) can be verified correct while the tunnel still fails. When there's no handshake and everything else checks out, audit the key placement — and trust `sudo wg show`, which displays the live peer keys, over what any config file or app screen claims.



\## Verification



\- `sudo wg show` on the server lists the client's public key as the peer with a recent `latest handshake:`

\- Phone on cellular (WiFi off) reaches the Proxmox web UI through the tunnel

\- Remote client DNS queries appear in the Pi-hole query log (confirming the DNS tie-in works over the tunnel)

\- Only UDP 51820 is forwarded; exposure documented and limited to the WireGuard listener



\## Current State / Next Steps



\- ✅ Tunnel working end-to-end from cellular; verified handshake and traffic

\- ✅ Client DNS routed through Pi-hole for filtered resolution over the tunnel

\- 🔜 DuckDNS auto-updater (cron) — until this exists, a public-IP change silently breaks remote access until the record is updated manually; flagged in the README's "What I'd Do Differently"

\- 🔜 Additional client devices (laptop) as needed

