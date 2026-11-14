# Home Network Target Architecture v0.4-simple

> **STATUS: CURRENT TARGET / PLANNING**
> **Version:** v0.4-simple
> **Date:** 2026-11-14
> **Previous design:** [v0.3](network-v0.3.md), frozen historical design

## Why v0.4 exists

v0.3 was intentionally more segmented: six active VLANs, including a dedicated Media VLAN for a single device and a dedicated Management VLAN. v0.4-simple reduces that complexity while keeping the security boundaries and VPN-routing behaviours that are needed now.

Media and Management no longer need dedicated VLANs initially. They can be introduced later if the homelab grows.

## Networks

Four VLANs are active. VLAN 40 is reserved.

| VLAN | Name | Subnet | Internet egress | SSID |
|---|---|---|---|---|
| 10 | Home | 10.10.10.0/24 | Mullvad via OPNsense (fail closed). Ugoos AM9 Pro: Proton, see below | Home |
| 30 | Direct | 10.10.30.0/24 | ISP WAN, no commercial VPN | Direct |
| 40 | IoT | 10.10.40.0/24 (reserved) | none | none |
| 50 | Infrastructure | 10.10.50.0/24 | POLICY TBD / per service | none |
| 60 | Guest | 10.10.60.0/24 | Mullvad via OPNsense (fail closed) | Guest |

Gateway addresses are not assigned yet. Only the /24 subnets are documented.

### VLAN 10 - Home

Trusted household devices. Default Internet egress is Mullvad via OPNsense.

Examples:
- GrapheneOS personal phone
- Desktop PC
- MacBook and other laptops
- residents' trusted devices
- dual-use work laptop in personal mode
- Ugoos AM9 Pro

Special case, Ugoos AM9 Pro:
- It receives a DHCP reservation.
- OPNsense policy-routes it through Proton VPN instead of Mullvad.
- It stays wired, on an untagged/access VLAN 10 port.
- The TV connects only to the Ugoos over HDMI and stays offline.

SSID: Home.

### VLAN 30 - Direct

Internet egress is the ISP WAN, without a commercial VPN.

Purpose:
- work and corporate VPN
- websites that block Mullvad
- VPN troubleshooting
- temporary direct Internet access

SSID: Direct.

At the desk:
- The managed desk switch receives a VLAN 10 + VLAN 30 tagged uplink.
- The Desktop normally uses VLAN 10.
- The docking station and work laptop use VLAN 30.
- The mechanism that gives the Desktop convenient VLAN 30 access is TBD.

### VLAN 40 - IoT

Subnet 10.10.40.0/24 is reserved. Status: **RESERVED / INACTIVE**.

Do not deploy DHCP, an SSID or firewall configuration for this VLAN yet. Activate it only when actual IoT or smart-home devices justify it.

### VLAN 50 - Infrastructure

Internet egress: **POLICY TBD / per service**. No SSID.

This VLAN combines the previous Servers (VLAN 50) and Management (VLAN 99) networks initially.

Contains:
- OPNsense management
- main managed switch management
- desk managed switch management
- Flint 2 management
- Slate Plus management
- Beelink / Proxmox
- Synology DS218
- Linkwarden (currently on the MacBook, planned migration)
- Jellyfin
- Floppy
- Sonarr
- Radarr
- Prowlarr
- SABnzbd
- Grafana / Prometheus / observability
- future homelab services

Security:
- VLAN 10 clients may reach selected user-facing services such as Jellyfin and Linkwarden.
- Infrastructure administration interfaces are not generally exposed to all VLAN 10 clients.
- ADMIN_DEVICES should later identify the Desktop PC and the MacBook/admin laptop and allow those devices to reach administrative interfaces.
- Synology DSM administration is restricted in the same way.
- Exact VLAN 50 Internet egress remains TBD per service.

### VLAN 60 - Guest

Internet egress: Mullvad. SSID: Guest.

Policy:
- Internet allowed.
- Home denied.
- Direct denied.
- Infrastructure denied.
- Future IoT denied.
- Wireless client isolation enabled.
- The VPN fails closed, with no ISP fallback.

## Wi-Fi

Two wired access points:
1. Existing Flint 2 / GL-MT6000.
2. Planned GL.iNet Slate Plus / GL-A1300, planned specifically as an AP/bridge.

After OPNsense is deployed, both operate as access points/bridges, not routers. DHCP, NAT and routing are disabled on both.

Both broadcast exactly these SSIDs, with the same names, security settings and passwords:

| SSID | VLAN |
|---|---|
| Home | 10 |
| Direct | 30 |
| Guest | 60 |

Both AP uplinks are 802.1Q tagged-only trunks carrying VLAN 10, 30, 50 and 60, with no native/untagged VLAN. VLAN 50 is the AP management network.

No Media, Infrastructure/Management or IoT SSID is active.

## Routing, DHCP and DNS

OPNsense is the only:
- router
- firewall
- DHCP server
- local DNS server
- VLAN gateway
- commercial VPN gateway

Flint 2 and Slate Plus have DHCP off and routing/NAT off.

DHCP for VLAN 10, 30, 50 and 60 is served by OPNsense.

## VPN policy

The ISP WAN is the only physical Internet path. Mullvad and Proton are logical WireGuard tunnels that run over the ISP WAN. They are not separate WAN connections.

| Network | Internet path | If the VPN is down |
|---|---|---|
| Home (VLAN 10) | Mullvad by default | Internet blocked, no ISP fallback |
| Ugoos AM9 Pro (on VLAN 10) | Proton, through a device-specific OPNsense policy route | Internet blocked, no ISP fallback. Local services stay reachable |
| Direct (VLAN 30) | ISP WAN, no commercial VPN | Independent of Mullvad and Proton |
| Guest (VLAN 60) | Mullvad | Internet blocked, no ISP fallback |
| Infrastructure (VLAN 50) | POLICY TBD / per service | not decided |

Local inter-VLAN traffic does not need to travel through Mullvad or Proton.

## Physical topology

```
Internet
  -> ISP modem/ONT
  -> OPNsense
  -> managed 2.5GbE core switch
```

### Trunks (802.1Q, tagged)

| Link | Tagged VLANs | Native/untagged VLAN |
|---|---|---|
| OPNsense to core switch | 10, 30, 50, 60 | none |
| Core switch to Flint 2 | 10, 30, 50, 60 | none |
| Core switch to Slate Plus | 10, 30, 50, 60 | none |
| Core switch to desk managed switch | 10, 30 | none |

### Access ports (untagged)

| Device | VLAN |
|---|---|
| Ugoos AM9 Pro | 10 |
| Synology DS218 | 50 |
| Beelink / Proxmox (initially) | 50 |
| Desk: Desktop PC | 10 |
| Desk: docking station / work laptop | 30 |
| Desk: spare ports | 10 or 30 as useful |

The v0.3 Beelink trunk (VLAN 50 + 99) no longer applies. No other Proxmox VLAN membership is defined yet.

## Hardware status

| Status | Hardware |
|---|---|
| Existing / reused | ISP modem/ONT, Flint 2 / GL-MT6000, Beelink SER (Ryzen 7 3750H, 16 GB, 512 GB), Synology DS218 (2 x 2 TB, mirrored), Ugoos AM9 Pro, TV (deliberately offline), Desktop PC, MacBook, GrapheneOS phone, dual-use work/personal laptop and dock |
| Planned / to buy | Dedicated OPNsense appliance, managed 2.5GbE core switch, managed desk switch, GL.iNet Slate Plus / GL-A1300 |
| Future / optional | DIY/Jonsbo media NAS, VLAN 40 IoT activation |

The OPNsense appliance and the managed switches have no chosen models yet.

## Changes from v0.3

- VLAN 20 Media is deferred/removed from active deployment.
- The Ugoos moves into VLAN 10 with device-specific Proton routing.
- VLAN 99 Management is deferred/removed from active deployment.
- Servers and management are initially combined into VLAN 50 Infrastructure.
- The active VLAN count drops from six to four.
- The active SSIDs remain Home, Direct and Guest.
- The Slate Plus is added as the second wired AP.
- VLAN 40 remains reserved.

## Why this is simpler

- Fewer VLANs: four active instead of six.
- Fewer trunk memberships: the Flint 2 and Slate Plus trunks carry the same four VLANs, and the Beelink and Synology are plain access ports.
- No separate management network initially.
- No dedicated Media network for a single Ugoos device.
- Only three SSIDs across both APs.
- One DHCP, DNS and router authority: OPNsense.
- Media VLAN 20 and Management VLAN 99 can be reintroduced later without redesigning the whole network.

## Unresolved items

- **Management/recovery access design:** TBD before deployment. v0.3 had a dedicated VLAN 99 MGMT / RECOVERY port. VLAN 99 is deferred in v0.4-simple, so there is no recovery path defined yet.
- **VLAN 50 Internet egress:** POLICY TBD / per service.
- **Desktop access to VLAN 30:** the mechanism for convenient access is not chosen.
- **Remote GrapheneOS / private-home access:** still under test, not part of the finalized topology.
- **Hardware selection:** the OPNsense appliance and the managed switches are not chosen.
- **Gateway addresses:** not assigned yet.
- **ADMIN_DEVICES:** to be defined later, with the Desktop PC and the MacBook/admin laptop as the intended members.

## Secrets

This document contains no keys, passwords, tokens, certificates or VPN configuration files.
