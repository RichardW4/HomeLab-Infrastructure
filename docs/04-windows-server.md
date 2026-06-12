\# Windows Server 2025 VM — Future Domain Controller



> Part of \[HomeLab-Infrastructure](../README.md). This doc covers the Windows Server VM through its Phase 1 end-state: installed, reachable, and deliberately unconfigured. The Active Directory build it exists for is Phase 3 work. Its problem entries include the project's best example of a failure mode recurring after it was "solved."



\## Purpose



Windows Server 2025 will become the Active Directory domain controller for the isolated lab network — the centerpiece of the Phase 3 identity work (AD DS, DNS, domain-joined clients) and the realistic enterprise target the Kali machine exists to practice against. Most real-world environments this lab imitates are AD environments; a security lab without one is missing the thing attackers actually go after.



\## Design Decisions



\- \*\*`vmbr1` placement (isolated lab network).\*\* The domain controller serves the \*lab's\* identity, not the household's — and as the eventual practice target, it belongs behind pfSense with the attack machine, never on the home LAN.

\- \*\*8 GB RAM.\*\* Comfortable headroom for the DC role later (AD DS + DNS idle well under that); confirmed against real in-guest usage rather than the hypervisor's allocation graph (see Problems).

\- \*\*Built in Phase 1, configured in Phase 3.\*\* Same rationale as pfSense: building the VM early captures it in the first clean-install snapshot and the daily backup job. The roles (AD DS, DNS) wait until the phase that needs them — a snapshot of a clean, unpromoted server is the ideal rollback point before AD experimentation begins.

\- \*\*Zero roles configured is the intended current state\*\*, not an omission. Server Manager launches at every logon and sits unconfigured by design.



\## Build Steps



\### 1. Create the VM

Standard Proxmox VM on `vmbr1`, 8 GB RAM, with the Windows Server 2025 installer ISO attached (plus a second drivers ISO during install).



\### 2. Install

Ran the installer to a signed-in desktop. The install path hit the boot-order race documented below — the fix (controlling boot order) is part of the canonical build procedure now, not a workaround.



\### 3. Post-install cleanup — detach all install media

Removed both ISOs from the VM's Hardware tab after the install was confirmed working. This is a required step, not housekeeping — see the second problem entry for what happens when it's skipped.



\### 4. Network

The VM pulls a DHCP lease from pfSense on the lab subnet (`10.10.10.x`) — fine for an idle server, \*\*not\*\* acceptable for a domain controller. A static address on the lab subnet is a hard prerequisite of the Phase 3 promotion (clients resolve the domain via the DC's DNS; a DC whose address can move breaks the domain).



\## Problems Encountered



\### Black screen at boot — the installer keypress race



\*\*Symptom:\*\* The freshly-created VM hung on a black screen or dropped into a recovery menu instead of starting the Windows installer.



\*\*Root cause:\*\* The "Press any key to boot from CD or DVD" prompt lasts under a second. Missing it sends the VM to the empty hard disk, which has nothing to boot.



\*\*Fix:\*\* Temporarily disabled the hard disk in the VM's boot order, leaving only the CD/DVD — the installer then boots deterministically with no key-press race. Re-enabled the disk after installation.



\*\*Lesson:\*\* Don't compete with a sub-second prompt; control the boot order instead. Deterministic beats fast-fingers every time.



\### The same trap, six days later — "broken" VM that was never broken



\*\*Symptom:\*\* Days after a successful install, the VM appeared dead — it wouldn't boot to Windows, and got written off as broken and parked for repair.



\*\*Root cause:\*\* The installer ISO was still attached and the CD still preceded the disk in boot order. A stray keypress during the boot prompt relaunched the \*installer\*, which looks exactly like a broken machine if you're not expecting it. The original problem was solved; the conditions that caused it were left in place.



\*\*Fix:\*\* Completed the boot into the installed OS, then \*\*detached both ISOs from the VM hardware\*\* — removing the conditions, not just the symptom. Clean boots confirmed afterward.



\*\*Lesson:\*\* A fix that leaves the failure's preconditions intact isn't finished. "Re-enable the disk after installation" should have been "and remove the install media" — the trap stayed armed for six days because cleanup was treated as optional. The incident report structure exists precisely to catch this: if the fix doesn't change the conditions in the root cause, it will recur.



\### Hypervisor showed 8.05 GB of 8 GB used at idle



\*\*Symptom:\*\* Proxmox's summary graph showed the VM consuming its entire memory allocation (slightly more, in fact) while Windows sat idle at the desktop.



\*\*Diagnosis:\*\* Checked inside the guest — Task Manager showed \*\*1.7 GB\*\* actually in use. The hypervisor and the guest were measuring different things.



\*\*Root cause:\*\* Without the QEMU guest agent installed in Windows, Proxmox has no visibility inside the guest, so its graph reports host-side \*allocation\*. Windows touches all of its RAM at boot, so the full 8 GB is resident from the host's perspective immediately and permanently. The extra 0.05 GB is normal QEMU device-emulation overhead.



\*\*Fix:\*\* None needed — the correct response was verifying the real number in-guest, not resizing the VM. Installing the QEMU guest agent + VirtIO balloon driver (from the VirtIO drivers ISO) is logged as a future nicety: it gives Proxmox true in-guest stats, clean shutdowns, and IP reporting.



\*\*Lesson:\*\* Hypervisor memory graphs for Windows guests show allocation, not usage, until a guest agent bridges the gap. Check inside the guest before concluding a VM needs more RAM — "free memory is wasted memory" is Windows working as designed.



\## Honest Scope Notes



\- \*\*This is not a domain controller yet.\*\* It's a clean, signed-in Server 2025 install with zero roles. Claiming "Active Directory" before `AD DS` is promoted would be exactly the resume-inflation this repo avoids; what exists today is the prepared foundation.

\- \*\*Managed via the Proxmox console\*\* — the Windows exception to the lab's SSH management model (Windows guests use console now, RDP within the lab network later).



\## Verification



\- VM boots directly to the installed OS with no install media attached

\- Sign-in to the desktop works; Server Manager loads

\- In-guest memory at idle \~1.7 GB, confirming the 8 GB allocation is ample

\- Holds a DHCP lease from pfSense on the lab subnet



\## Current State / Next Steps



\- ✅ Server 2025 installed on `vmbr1`, signed in, install media detached, boot trap closed

\- ✅ Captured in the clean-install snapshot and daily vzdump job

\- 🔜 QEMU guest agent + VirtIO balloon driver (accurate stats, clean shutdowns) — bundle with the next session touching this VM

\- 🔜 Pre-promotion checklist before Phase 3: set a static IP on the lab subnet (outside the pfSense DHCP pool) and finalize the hostname — both are painful to change \*after\* a DC is promoted, trivial before

\- 🔜 Phase 3: AD DS + DNS roles, domain creation, and the identity lab this VM exists for

