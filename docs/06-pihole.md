# Pi-hole — Network DNS Sinkhole / Ad Blocker

> Part of [HomeLab-Infrastructure](../README.md). This doc covers the Pi-hole LXC: install, blocklist configuration, the rollout strategy decision, and a DNS-over-HTTPS bypass investigation that turned into the most instructive debugging session of the project so far.

## Purpose

Pi-hole is a DNS sinkhole: every device pointed at it sends DNS queries through it, and queries for known ad/tracker/malware domains get answered with a null address instead of the real one — the request dies before it ever leaves the network. Beyond household ad-blocking, this is the same mechanism SOC teams use to neutralize malware command-and-control domains (sinkhole the C2 domain and infected hosts can't phone home), which is why this was worth building as a security exercise and not just a convenience.

## Design Decisions

- **LXC, not a VM or Docker.** Pi-hole is a tiny single-purpose service — the same lightweight-container pattern as the WireGuard deployment. 1 core, 1 GB RAM, 8 GB disk on `vmbr0`. No Docker dependency keeps Phase 2's container work cleanly separated.
- **Reserved IP, non-negotiable.** The container holds a DHCP reservation at `192.168.1.87`. A DNS server whose address drifts silently breaks name resolution for every device pointing at it — this is the one service where a pinned address is mandatory, not optional.
- **Per-device DNS rollout instead of network-wide.** The intended design was to set Pi-hole as the DNS server in the gateway's DHCP settings so every device inherits it automatically. The AT&T gateway's Subnets & DHCP page exposes **no editable DNS field** — it forces itself as the distributed DNS server, full stop. That left two options: run Pi-hole as the network's DHCP server (rejected — the cutover window would interrupt connectivity for an always-on, multi-user household), or point devices at `192.168.1.87` individually. Per-device is manual and doesn't cover new/guest devices by default, but it's zero-downtime and fully reversible. Right call for this network; wrong call for a network you fully control.
- **Blocklists: StevenBlack + OISD + Hagezi** (~680k domains combined). Deliberately stopped there — over-stacking lists inflates false positives faster than it improves blocking.

## Build Steps

### 1. Create the LXC
Downloaded the Ubuntu CT template (**local → CT Templates** — LXCs use CT templates, not install ISOs), then **Create CT**: 1 core, 1 GB RAM, 8 GB disk, network on `vmbr0`.

### 2. Reserve the IP
Created a DHCP reservation on the AT&T gateway binding the container's MAC to `192.168.1.87` before doing anything else, so every later configuration step references an address that can't move.

### 3. Install Pi-hole
Ran the official installer inside the container:

```bash
curl -sSL https://install.pi-hole.net | bash
```

The text-menu walkthrough covers upstream DNS selection and confirms the static address. The web admin UI comes up at `http://192.168.1.87/admin`.

### 4. Add blocklists
Added OISD and Hagezi alongside the default StevenBlack list under **Adlists**, then rebuilt gravity. ~680k domains loaded.

### 5. Test on one device before any rollout
Pointed a single Windows PC at Pi-hole manually (adapter settings → IPv4 DNS = `192.168.1.87`, secondary left **blank**, then `ipconfig /flushdns`) and confirmed blocking on ad-heavy sites before touching any other device. This ordering mattered — see Problems below.

### 6. Per-device rollout
Rolling devices onto `192.168.1.87` one at a time. Reliability rules learned along the way:

- **Leave the secondary/alternate DNS blank.** A public fallback (8.8.8.8 etc.) lets the device intermittently bypass Pi-hole and makes blocking look randomly broken.
- **Disable IPv6 DNS where the device exposes it.** This Pi-hole is IPv4-only; a leftover IPv6 resolver is a silent bypass path.
- **Verify by traffic, not by settings.** A device isn't "on Pi-hole" until its IP shows real browsing queries in the Query Log. Smart TVs in particular often hardcode `8.8.8.8` or use DNS-over-HTTPS and ignore the configured DNS entirely.

### 7. WireGuard tie-in
Updated each WireGuard client config's `[Interface] DNS =` line from the gateway (`192.168.1.254`) to Pi-hole (`192.168.1.87`), so ad/tracker blocking follows remote clients through the tunnel. Client-side change only — the server's `wg0.conf` is untouched, and the existing `AllowedIPs = 192.168.1.0/24` already routes to `.87`.

## Problems Encountered

### Dashboard showed queries arriving but 0 blocked — client was bypassing Pi-hole via encrypted DNS

**Symptom:** With 680k+ domains loaded, the test phone generated queries visible on the dashboard, but the blocked count stayed at **zero** — even on ad-heavy news sites. Tuning blocklists, groups, and client settings changed nothing.

**Diagnosis path:**
1. **Query Log status column** — every entry read "OK (forwarded/cached)," none "Blocked (gravity)." So Pi-hole was answering queries, just never blocking ones.
2. **The tell:** the *only* traffic from the phone's IP was `www.google.com` every 30 seconds — Android's connectivity heartbeat. The actual ad/tracker lookups from browsing never appeared at all. The phone was sending its heartbeat to Pi-hole while resolving real browsing DNS somewhere else.
3. **Server-side isolation test:** on the container itself, `nslookup flurry.com 127.0.0.1` returned `0.0.0.0` — proving Pi-hole's blocking engine worked and the problem was entirely client-side.
4. **Clean-client test:** pointed a Windows PC at `192.168.1.87` (IPv4 only, secondary blank, `ipconfig /flushdns`) — ads immediately started blocking.

**Root cause:** The phone was resolving browsing DNS over **DNS-over-HTTPS**, routing around the network's DNS server entirely. Private DNS and Chrome Secure DNS were off, meaning the bypass lived at the OS/app level — mobile encrypted DNS is sticky and doesn't respect the network-assigned resolver.

**Fix:** Validated the deployment on the PC and proceeded with the rollout; the phone's bypass is OS-level behavior, not a Pi-hole misconfiguration. The complete fix — forcing DNS at the gateway and blocking outbound DoH — is logged as a future hardening step (and partially impossible on this gateway anyway).

**Lessons:**
- If a client shows queries but zero blocks, check *which* queries are reaching the resolver. Background heartbeats reaching Pi-hole while browsing traffic doesn't is the signature of an encrypted-DNS bypass.
- `nslookup <known-bad-domain> 127.0.0.1` on the server cleanly splits "Pi-hole is broken" from "the client is bypassing" in one command.
- This exact problem — DoH defeating DNS-based monitoring and filtering — is a live headache for SOC, network, and content-filtering teams in production. Hitting it firsthand on a homelab is the same investigation at smaller scale.

## Honest Scope Notes

- **Block percentage is naturally low** (single digits) because most DNS traffic is legitimate. Judge effectiveness by Top Blocked Domains and absolute counts, not the percentage.
- **In-stream video ads are not blockable by DNS.** YouTube/Hulu/ad-tier streaming serve ads from the same domains and CDNs as the content — sinkholing the domain would sinkhole the show. DNS blocking handles display ads, trackers, and telemetry; it is the wrong tool for stream-injected ads, and pretending otherwise is how Pi-hole gets oversold.
- **Single point of failure:** every device pointed at Pi-hole loses DNS if the container is down. Acceptable here (snapshot + backup exist, restore is minutes); in production this is why resolvers are deployed in pairs.

## Verification

- `nslookup flurry.com 127.0.0.1` on the container returns `0.0.0.0` (blocking engine confirmed)
- Test PC shows browsing-pattern queries in the Query Log with "Blocked (gravity)" entries
- Ad-heavy pages render without display ads on rolled-out devices
- WireGuard client connected from cellular resolves through `.87` (remote queries visible in the log)

## Current State / Next Steps

- ✅ Pi-hole LXC live at `192.168.1.87` (reserved), 680k-domain blocklist, blocking confirmed
- ✅ WireGuard clients' DNS pointed at Pi-hole
- 🔄 Per-device rollout in progress (PC and TV done; remaining devices gradual)
- 🔜 Future hardening: block outbound DoH/known encrypted-DNS endpoints at the firewall so clients can't route around the resolver