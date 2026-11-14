# GrapheneOS WG Tunnel with Mullvad

**Status:** current KNOWN-GOOD configuration. Tested on 2026-11-14.

## Purpose

- GrapheneOS personal phone.
- WG Tunnel using a Mullvad WireGuard configuration.
- Mobile data automatically uses Mullvad.
- Wi-Fi VPN usage is manual.
- No SSID detection, location permission, Shizuku, or app split tunneling.

## Settings

### WG Tunnel

| Setting | Value |
|---|---|
| App mode | VPN |
| Tunnel on Wi-Fi | OFF |
| Tunnel on mobile data | ON |
| Preferred mobile tunnel | `de-ber-wg-103` |
| Tunnel on Ethernet | OFF |
| Stop on no network | OFF |
| Auto-tunnel start on boot | ON |
| App split tunneling | OFF |
| Remote control | OFF |

### Android / GrapheneOS

| Setting | Value |
|---|---|
| Always-on VPN | OFF |
| Block connections without VPN | OFF |
| Location permission for WG Tunnel | NOT GRANTED |

## Verified behavior

| Test | Result |
|---|---|
| Mobile data activates Mullvad automatically | PASS |
| Switching to Wi-Fi stops the automatic tunnel | PASS |
| Mullvad can be manually enabled while on Wi-Fi | PASS |
| Mullvad IPv4/IPv6 connectivity | PASS |
| DNS leak test | PASS |
| WebRTC leak test | PASS |
| Reboot/profile startup on mobile starts the VPN correctly | PASS |
| Wi-Fi startup leaves the VPN off | PASS |
| Failure test with an intentionally unreachable WireGuard endpoint (`1.1.1.1:1`): handshake failed and Internet traffic was blocked rather than falling back to mobile carrier Internet | PASS |

## Security Notes

- Do not commit the actual Mullvad WireGuard `.conf`. It contains private key material.
- Store the exported `.conf` separately in encrypted storage.
- This repository documents settings and behavior only.
- `de-ber-wg-103` is the known-good profile name, not a secret.

## Design rationale

- Location-based Wi-Fi automation was deliberately rejected.
- Wi-Fi VPN control remains manual.
- Mobile automation provides protection when leaving Wi-Fi.
- This avoids enabling system Location services, granting location permission, running Shizuku, or using app-level split tunneling.
