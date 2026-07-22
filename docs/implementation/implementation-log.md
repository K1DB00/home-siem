# Home SIEM Implementation Log

## Objective

This document records the chronological implementation history of the Home SIEM laboratory: what was actually built, in what order, how each step was validated, and how any issues encountered along the way were diagnosed and resolved. It is a build record, not a design document — the intended architecture, rationale, and long-term specifications live in `docs/architecture/`. Where an implementation detail deviates from what an architecture document describes, that deviation is called out explicitly rather than silently reconciled, since the architecture documents are not modified as part of this log.

---

## Phase 1 – Windows Endpoint

The Windows endpoint (`CYBERLAB-WIN11`) was the first component built in the lab.

- Created a new virtual machine in VMware Workstation Pro using a Windows 11 Enterprise Evaluation installation image.
- Configured virtual hardware per the design baseline: 4 vCPU (1 virtual processor × 4 cores), 8 GB RAM, a 64 GB NVMe virtual disk (thin provisioned), UEFI firmware, virtual TPM enabled, and Secure Boot enabled.
- Completed the Windows 11 guest OS installation.
- Installed VMware Tools inside the guest to enable proper display scaling, clipboard integration, and host-guest time synchronization.
- Applied all available Windows Updates until the guest reported fully up to date.
- Took a baseline snapshot, "01 - Clean Windows Baseline", capturing the VM immediately after OS installation, VMware Tools, and Windows Update, before any further hardening or configuration.
- Configured a static IP address on the VMnet1 (host-only) network adapter for internal lab communication, leaving the gateway and DNS fields empty on that interface as intended for a host-only segment.

## Phase 2 – Host-only Network Validation

With static addressing in place, connectivity between the host and `CYBERLAB-WIN11` over VMnet1 was validated.

- Initial testing found that the host could not ping the Windows endpoint over the VMnet1 host-only network, despite the interface being correctly addressed.
- **Root cause:** Windows Defender Firewall blocks inbound ICMP echo requests by default on a fresh Windows 11 installation. No inbound rule permitted the ping traffic to reach the guest.
- **Resolution:** Enabled the built-in "File and Printer Sharing" firewall rule group in Windows Defender Firewall, which includes the inbound ICMPv4/ICMPv6 echo-request rules. Windows Defender Firewall itself was left enabled throughout — no rule group beyond this one was disabled or removed, and no blanket allowance for ICMP was created.
- **Final validation:** Bidirectional ping connectivity between the host (`192.168.72.1`) and `CYBERLAB-WIN11` (`192.168.72.20`) was confirmed successful in both directions.

## Phase 3 – Ubuntu Server

The Ubuntu endpoint (`CYBERLAB-UBUNTU`) was built next, following the same VM-first, network-second sequence used for the Windows endpoint.

- Created a new virtual machine in VMware Workstation Pro using an Ubuntu Server 24.04 LTS installation image.
- Configured virtual hardware: UEFI firmware with Secure Boot enabled, 2 vCPU, 4 GB RAM, and a 40 GB thin-provisioned virtual disk.

  > **Note on deviation from architecture documentation:** `docs/architecture/03-vm-specifications.md` (Section 5) documents a planned allocation of 2 GB RAM and a 32 GB disk for this VM. The VM was actually built with 4 GB RAM and a 40 GB disk, as recorded above. This log reflects what was implemented; reconciling the architecture document's planned figures with the as-built configuration is tracked as follow-up work and is out of scope for this log, which does not modify architecture documents.

- Installed OpenSSH Server during guest OS setup to enable remote administration from the host.
- Configured a static IP address on the VMnet1 (host-only) interface for internal lab communication, consistent with the addressing approach used for the Windows endpoint.
- Took a baseline snapshot capturing the VM immediately after OS installation and initial network configuration.

## Phase 4 – Ubuntu Network Configuration

VMnet1 addressing on `CYBERLAB-UBUNTU` was configured manually through Netplan rather than relying on any default configuration.

- Edited the Netplan YAML configuration to assign a static IPv4 address to the VMnet1 interface, with no gateway or DNS server configured on that interface — matching the host-only segment's design.
- Applied the Netplan configuration and confirmed the static address (`192.168.72.30/24`) was active on the interface.
- Validated Host ↔ Ubuntu connectivity: the host was able to reach `CYBERLAB-UBUNTU` over VMnet1, and the reverse direction was confirmed as well.
- Validated Windows ↔ Ubuntu connectivity: `CYBERLAB-WIN11` and `CYBERLAB-UBUNTU` were confirmed able to reach each other directly over the VMnet1 host-only segment.

## Phase 5 – Internet Connectivity

At this point `CYBERLAB-UBUNTU` had only a single network interface (VMnet1, host-only) and no path to the internet for package updates. A second interface was added to provide outbound connectivity without altering the host-only segment already in use.

- Added a second virtual network adapter to the `CYBERLAB-UBUNTU` VM in VMware Workstation Pro, connected to VMnet8 (NAT).
- After booting the VM, found that the new interface was not automatically configured — Netplan only manages interfaces explicitly declared in its YAML configuration, so the newly attached adapter had no address and was not in use.
- Manually added a second interface definition to the Netplan configuration for the new adapter, configured for DHCP.
- Applied the updated configuration and confirmed the interface successfully acquired an address from the VMware NAT DHCP service.
- Verified the default route was installed via the new NAT interface, while the VMnet1 interface remained gateway-less as designed.
- Confirmed outbound connectivity by reaching an external host over the internet through the NAT interface.

This established the lab's final dual-network pattern on `CYBERLAB-UBUNTU`, matching the design already in place for `CYBERLAB-WIN11`:

- **VMnet1 (host-only)** for internal lab communication — telemetry, agent management, and host-to-VM traffic.
- **VMnet8 (NAT)** for outbound internet access — OS and package updates only.

## Phase 6 – Base System Preparation

With networking complete on both interfaces, `CYBERLAB-UBUNTU` was brought up to a working baseline.

- Ran a full system update (`apt update` / `apt upgrade`) over the NAT interface established in Phase 5.
- Installed Open VM Tools to enable proper guest integration under VMware Workstation Pro (equivalent in role to VMware Tools on the Windows endpoint).
- Installed a small set of basic administration and diagnostic utilities to support ongoing lab work and connectivity troubleshooting.
- Took an updated snapshot representing the current Ubuntu baseline: fully patched, dual-homed (VMnet1 static + VMnet8 DHCP), with Open VM Tools and base utilities installed.

---

## Validation Summary

| Validation | Result | Notes |
|---|---|---|
| Windows boot validation | Pass | `CYBERLAB-WIN11` installs and boots cleanly with UEFI, virtual TPM, and Secure Boot enabled |
| Ubuntu boot validation | Pass | `CYBERLAB-UBUNTU` installs and boots cleanly with UEFI and Secure Boot enabled |
| Host ↔ Windows | Pass | Required enabling the "File and Printer Sharing" firewall rule group before ICMP succeeded (Phase 2) |
| Host ↔ Ubuntu | Pass | Validated after static Netplan configuration of the VMnet1 interface (Phase 4) |
| Windows ↔ Ubuntu | Pass | Direct connectivity confirmed over VMnet1 (Phase 4) |
| Host-only networking (VMnet1) | Pass | Static addressing confirmed on both endpoints; no gateway or DNS present on this segment, as designed |
| NAT connectivity (VMnet8) | Pass | Required a manually added Netplan interface on Ubuntu before DHCP was acquired (Phase 5) |
| VMware Tools (Windows) | Pass | Installed and functional on `CYBERLAB-WIN11` |
| Open VM Tools (Ubuntu) | Pass | Installed and functional on `CYBERLAB-UBUNTU` |

---

## Lessons Learned

- **VMware host-only subnets are installation-specific.** The VMnet1 subnet used in this lab (`192.168.72.0/24`) and the VMnet8 NAT subnet assigned automatically by VMware are values specific to this host's VMware installation, not fixed defaults — they should be verified in VMware's Virtual Network Editor rather than assumed when reproducing the lab elsewhere.
- **Windows Defender Firewall blocks inbound ICMP by default.** A fresh Windows 11 installation does not respond to ping until the appropriate built-in firewall rule group is enabled. This is a targeted, minimal change rather than a reason to weaken the firewall's overall posture.
- **Ubuntu Netplan only configures interfaces explicitly declared in its YAML.** Attaching a new virtual NIC to a VM does not make it usable until a corresponding interface definition is added to the Netplan configuration and applied — this applied to both the initial VMnet1 interface and the later VMnet8 interface.
- **Separating host-only and NAT traffic simplifies lab management.** Keeping internal lab communication (VMnet1) and internet-bound update traffic (VMnet8) on distinct interfaces made it straightforward to reason about and validate each connectivity path independently, and kept the internal segment free of a default gateway.

---

## Current Project Status

- [x] Architecture documentation
- [x] Windows endpoint
- [x] Ubuntu endpoint
- [x] Network validation
- [ ] Docker
- [ ] Elastic Stack
- [ ] Fleet Server
- [ ] Elastic Agents
- [ ] Kali Linux
- [ ] Detection engineering
- [ ] Dashboards
