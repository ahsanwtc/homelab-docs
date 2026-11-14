# Home Network Architecture v0.3

> **STATUS: FROZEN HISTORICAL DESIGN**
> **Version:** v0.3
> **Date:** 2026-11-14

v0.3 is the detailed, pre-simplification target architecture. It is preserved on purpose as a design baseline: the next architecture revision will simplify the VLAN design, and this document records what that revision starts from. Do not edit it. Changes go into a new version.

The design was modelled in CTRoadmap as "Home Network Target Architecture v0.3". The exported atlas is in [`ctroadmap/atlas-v0.3.json`](ctroadmap/atlas-v0.3.json) (63 tiles, 46 links, 11 zones).

## Hardware status

| Status | Hardware |
|---|---|
| Existing | ISP modem/ONT, GL.iNet Flint 2 (GL-MT6000), Beelink SER Mini PC (Ryzen 7 3750H, 16 GB, 512 GB NVMe), Synology DS218 (2 x 2 TB, mirrored), Ugoos AM9 Pro, TV (offline), desktop PC, MacBook, dual-use work/personal laptop, docking station, GrapheneOS phone, current unmanaged desk switch (to be retired/replaced) |
| Proposed / to buy | Dedicated OPNsense appliance, main managed 2.5GbE VLAN-aware switch, small managed desk switch |
| Future / optional | DIY/Jonsbo media NAS, IoT devices and VLAN 40 activation |

## VLANs

| VLAN | Name | Subnet | Gateway | Internet egress | Notes |
|---|---|---|---|---|---|
| 10 | Trusted | 10.10.10.0/24 | 10.10.10.1 | Mullvad (fail closed) | Personal and household clients |
| 20 | Media | 10.10.20.0/24 | 10.10.20.1 | Proton VPN (fail closed for Internet) | Ugoos AM9 Pro. Local Jellyfin stays reachable if Proton is down |
| 30 | Direct | 10.10.30.0/24 | 10.10.30.1 | ISP WAN directly | VPN-bypass and compatibility network, independent of VPN health |
| 40 | IoT | 10.10.40.0/24 (reserved) | not assigned | none | RESERVED / INACTIVE / FUTURE. Not deployed, not carried on any trunk, no active SSID |
| 50 | Servers | 10.10.50.0/24 | 10.10.50.1 | POLICY TBD / per-service | Application and service workloads. Not assigned to Mullvad, Proton or the ISP |
| 60 | Guest | 10.10.60.0/24 | 10.10.60.1 | Mullvad (fail closed) | Internet-only, wireless client isolation |
| 99 | Management | 10.10.99.0/24 | 10.10.99.1 | none | Infrastructure management only. No Management SSID |

## Wi-Fi

The Flint 2 acts as a VLAN-aware access point, not as the main router. Three SSIDs are active:

| SSID | VLAN |
|---|---|
| Home | 10 (Trusted) |
| Direct | 30 (Direct) |
| Guest | 60 (Guest) |

There is no Media, Servers, Management or IoT SSID. An IoT SSID is future only.

## Trunks and access ports

All trunks are tagged-only. None has a native/untagged VLAN. Access ports are untagged in their assigned VLAN.

| Link | Type | VLANs |
|---|---|---|
| OPNsense to main managed switch | Trunk (tagged) | 10, 20, 30, 50, 60, 99 |
| Main switch to Flint 2 | Trunk (tagged) | 10, 30, 60, 99 (VLAN 99 is Flint 2 management only) |
| Main switch to Beelink/Proxmox | Trunk (tagged) | 50, 99 |
| Main switch to desk managed switch | Trunk (tagged) | 10, 30 |
| Main switch to Ugoos AM9 Pro | Access (untagged) | 20 |
| Main switch to Synology DS218 | Access (untagged) | 50 |
| Main switch MGMT / RECOVERY port | Access (untagged) | 99 |
| Desk switch to desktop PC | Access (untagged) | 10 |
| Desk switch to dock / work laptop | Access (untagged) | 30 |
| Desk switch spare ports | Access (untagged) | one spare on 10, one spare on 30 |

## Design decisions

### Internet path and VPN

- The only physical Internet path is Internet, ISP modem/ONT, ISP WAN, OPNsense.
- Mullvad WireGuard and Proton WireGuard are logical tunnels that originate on OPNsense and run over the ISP WAN. They are not separate WAN connections.
- VLAN 30 uses the ISP WAN directly, with no commercial VPN.
- VPN-routed VLANs fail closed, with no fallback to the ISP:
  - VLAN 10: Mullvad down means Internet blocked.
  - VLAN 60: Mullvad down means Internet blocked.
  - VLAN 20: Proton down means Internet blocked. Local Jellyfin access continues.
  - VLAN 30: always the ISP WAN, independent of Mullvad and Proton health.
- Local inter-VLAN traffic does not travel through Mullvad or Proton.

### Media

- The Ugoos AM9 Pro is wired, untagged/access VLAN 20, and reaches Jellyfin on VLAN 50.
- The TV connects to the Ugoos over HDMI and remains offline. It has no network connection.

### Servers

- The Beelink/Proxmox trunk carries VLAN 50 and VLAN 99 on purpose. VLAN 99 is Proxmox host management. VLAN 50 is application, VM and LXC workloads. The Proxmox/Linux bridge configuration is an implementation detail documented at deployment: architecture resolved, implementation deferred.
- The Synology DS218 is currently attached only to VLAN 50 (untagged/access), so DSM administration also happens on VLAN 50 for now. ADMIN_DEVICES are explicitly allowed to DSM on VLAN 50. Ordinary Trusted clients get no DSM administration access.
- Linkwarden currently runs on the MacBook. It is planned to move to Proxmox on VLAN 50.

### Management

- VLAN 99 holds infrastructure management only: OPNsense, main switch, desk switch, Flint 2 and Proxmox.
- A dedicated physical MGMT / RECOVERY port on the main managed switch is untagged/access VLAN 99. It is the emergency fallback for connecting a laptop if normal routed management access is broken.
- The desktop PC and the MacBook stay on VLAN 10 but are the only members of the ADMIN_DEVICES firewall alias. OPNsense explicitly allows ADMIN_DEVICES to reach VLAN 99. Other VLAN 10 clients, including the GrapheneOS phone and residents' devices, are denied. This is intentional, not a contradiction.

### Firewall intent (summary)

| Source | Allowed | Denied |
|---|---|---|
| VLAN 10 Trusted | Selected VLAN 50 user services, Internet via Mullvad. ADMIN_DEVICES also reach VLAN 99 and Synology DSM | VLAN 99 for all other clients |
| VLAN 20 Media | Jellyfin on VLAN 50, Internet via Proton | Trusted, Management, other servers unless explicitly required |
| VLAN 30 Direct | Internet via ISP | Internal networks by default (exceptions may be added later) |
| VLAN 50 Servers | Return traffic for established connections | New connections into client networks by default |
| VLAN 60 Guest | Internet via Mullvad, DNS and DHCP | All internal VLANs, including future VLAN 40 |
| VLAN 99 Management | Administrative infrastructure only | Ordinary household devices |

### DNS

OPNsense is the local DNS source. Local names use `home.arpa` (for example `jellyfin.home.arpa`, `proxmox.home.arpa`, `nas.home.arpa`, `opnsense.home.arpa`). `.local` is not used for ordinary DNS names.

### Desk

The desk has a managed desk switch that terminates both VLAN 10 (desktop) and VLAN 30 (docking station and work laptop). The work laptop uses VLAN 30 Direct over the dock for work and VLAN 10 over the Home SSID for personal use. The current unmanaged desk switch is replaced in the target and is not part of the traffic path.

## Unresolved items

- **VLAN 50 Internet egress:** per-service policy still to be decided.
- **Desktop switching between Trusted and Direct:** mechanism not chosen. Options are a second physical NIC or USB Ethernet, a switch-port change, or a VLAN-aware desktop configuration.
- **Remote GrapheneOS / private-home access:** under test. The current experiment uses WG Tunnel with Mullvad WireGuard: Home Wi-Fi turns the device VPN off, and outside the home Mullvad turns on automatically. Combined private-home WireGuard routing is not finalized and no solution (NetBird or otherwise) is selected.
- **Dedicated Synology VLAN 99 management:** future, to be decided.
- **Hardware selection:** the actual OPNsense appliance and the managed switches are not chosen.

## Restoring the CTRoadmap diagram

[`ctroadmap/atlas-v0.3.json`](ctroadmap/atlas-v0.3.json) is the full atlas, including tile notes explaining why each network exists. Import it through CTRoadmap, or send it with `PUT /api/atlas`. Zones are VLAN colored, existing hardware uses the live lifecycle, and proposed and future items use planned plus a title suffix, because CTRoadmap can't draw dashed lines or per-tile borders.

## Secrets

This document and the atlas export contain no keys, passwords, tokens, certificates or VPN configuration files.
