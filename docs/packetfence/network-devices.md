# PacketFence Network Device (NAS) Configuration

Every UniFi AP and switch that sends RADIUS to PacketFence is a RADIUS client (NAS). PacketFence must know each one, with the matching shared secret, before it will process its requests and map roles to VLANs.

> **Repository coverage is thin here.** The existing documentation records only the UniFi side and one requirement: *the shared secret must exactly match the secret configured for the corresponding UniFi NAS/network device in PacketFence.* The PacketFence network-device form itself was not captured. Fields below that the repository does not record are marked `to be captured`; no values have been invented.

## Where it lives

The Jumpcloud-Radius source screenshot shows the PacketFence navigation:

```text
Configuration
  -> Policies and Access Control
  -> Network Devices
       -> Switches
       -> Switch Groups
       -> Discover
```

UniFi APs and switches are defined under this section. Whether they are defined individually, by CIDR, or through a Switch Group in the tested system is `to be captured`.

## What must match the UniFi RADIUS profile

| PacketFence network device | UniFi RADIUS profile ([UniFi Configuration](../unifi/configuration.md)) | Status |
| --- | --- | --- |
| Device IP address / range | Source address of the AP or switch as seen by PacketFence | Tested requirement; exact values `to be captured` |
| RADIUS shared secret | Shared secret on both the authentication and accounting servers in the UniFi profile (`<RADIUS_SHARED_SECRET>`) | Tested requirement (must match exactly) |
| Authentication port | UDP/1812 | Tested |
| Accounting port | UDP/1813 | Tested |

## Fields to capture

Record these (with secrets redacted) next time the form is open:

| Field | Why it matters | Status |
| --- | --- | --- |
| Definition / IP or CIDR | Determines which UniFi devices PacketFence will accept RADIUS from | `to be captured` |
| Switch type / template (the UniFi or generic vendor type selected) | Controls which vendor attributes and disconnect method PacketFence uses | `to be captured` |
| Mode (production / testing / registration) | Determines whether PacketFence actually enforces returned VLANs | `to be captured` |
| RADIUS secret | Must equal the UniFi profile secret | Value never recorded; match requirement documented |
| Role -> VLAN mapping (role-by-VLAN-ID entries for `Staff`, `UVC`, student roles, etc.) | This is how a role chosen by an authentication rule becomes `Tunnel-Private-Group-Id`. Expected to be held on the network device or its group | `to be captured` |
| Default VLAN / registration / isolation VLANs | Behaviour when no role maps to a VLAN | `to be captured`; behaviour `needs validation` |
| Change-of-Authorization / disconnect settings | Whether PacketFence can re-authenticate or disconnect a session after a role change | `to be captured`; support on UniFi `needs validation` |
| Use of Switch Group | Whether settings are shared across all UniFi devices | `to be captured` |

## Checks

- PacketFence receives RADIUS from the device IP you defined (use `tcpdump` as in [Troubleshooting](../operations/troubleshooting.md)). A device not defined as a network device is expected to have its requests rejected or ignored; the exact symptom `needs validation`.
- Secret mismatch is a common cause of silent failure: confirm the same `<RADIUS_SHARED_SECRET>` on the UniFi authentication server, UniFi accounting server and the PacketFence device.
- Keep secrets out of the repository, screenshots and tickets ([Security Model](../security/security-model.md)).

## Suggested screenshot

`docs/images/pf-network-device-unifi.png` (switch/AP definition) and `docs/images/pf-network-device-roles.png` (role-to-VLAN mapping), both with secrets masked. Neither exists yet.
