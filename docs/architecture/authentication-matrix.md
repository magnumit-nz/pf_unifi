# Authentication Matrix

How each access method is authenticated, authorized and placed into a VLAN. Status uses the labels from the [overview](overview.md#how-to-read-this-documentation).

## Matrix

| | Wireless 802.1X | Wired 802.1X | UVC MAC-based authentication (MAB) | Unknown / unregistered MAC |
| --- | --- | --- | --- | --- |
| **Status** | Tested | Tested | Tested | `needs validation` (see below) |
| **Enforcement point** | UniFi AP | UniFi switch port (802.1X) | UniFi switch port, `802.1X Control = MAC-Based` | UniFi switch port (same) |
| **Client method** | PEAP/MSCHAPv2 (tested). TTLS is the profile default but was not the tested client method | PEAP/MSCHAPv2 (tested) | None: MAC address is the identity | None |
| **RADIUS to** | PacketFence UDP/1812, accounting UDP/1813 | Same | Same | Same |
| **PacketFence connection profile** | `Jumpcloud-BYOD` (filter: SSID) | `Jumpcloud_Wired` (filter: `Connection Type = Ethernet-EAP`) | Not applicable to the tested path; the RADIUS filter acts at the authorize stage | Not recorded |
| **Realm** | `NULL` (username without suffix) -> `Jumpcloud-Radius` | `NULL` -> `Jumpcloud-Radius` | Forced to `local` by filter | Not forced local: expected to stay `null` |
| **Credential check** | JumpCloud RADIUS via PacketFence proxy | JumpCloud RADIUS via PacketFence proxy | None. Not proxied | Expected to be proxied to JumpCloud RADIUS and fail (`needs validation`) |
| **Authorization source** | JumpCloud LDAP `memberOf` rule | JumpCloud LDAP `memberOf` rule | PacketFence local node record: registered MAC with category `UVC` | None expected |
| **Role** | Rule result (e.g. `Staff`) | Rule result | `UVC` | None expected |
| **VLAN** | Role's VLAN via tunnel attributes | Role's VLAN via tunnel attributes | `UVC` role's VLAN (example `200`) via tunnel attributes | No VLAN expected |
| **Auto-registration** | Enabled on profile | Enabled on profile | Registration is manual / administrative ([lifecycle](../operations/mab-device-lifecycle.md)) | Not applicable |
| **Trust basis** | Valid credentials + group membership | Valid credentials + group membership | Physical port + registered MAC + `UVC` category + restricted VLAN ([Security Model](../security/security-model.md#mab-trust-boundary)) | None |

## Request signatures

Observed attributes useful for telling the paths apart in RADIUS audit and `tcpdump`:

| Path | Observed / expected attributes |
| --- | --- |
| Wireless 802.1X | `EAP-Type = "PEAP"`, `Service-Type = "Framed-User"`, `Realm = "null"`, `User-Name = "testuser"` |
| Wired 802.1X | `NAS-Port-Id = "te1/0/14"`, `NAS-Port-Type = "Ethernet"`, `Service-Type = "Framed-User"`, `EAP-Type = "PEAP"` |
| MAB | MAC-shaped `User-Name` (e.g. `001122334455`); `Service-Type = Call-Check` is understood to identify MAB (diagnostic use only; not used as the policy trigger) |

Wired PEAP must not be mistaken for MAB at the authorize stage. A condition such as `connection_type == "Ethernet-NoEAP"` was observed to catch legitimate wired PEAP too early and is **not** used.

## Unknown and unregistered MAC behaviour

The repository establishes only the *design intent*: unknown MAB endpoints are not automatically trusted, because the local-handling filter needs `node_info.category == "UVC"`. The **observed outcome for a non-matching MAC has not been recorded.**

Expected behaviour from the configuration (`needs validation`):

1. The filter does not match, so `Realm` stays `null` and the request follows the JumpCloud proxy path.
2. JumpCloud RADIUS receives a MAC-shaped username it is not expected to know and is expected to reject it.
3. UniFi therefore keeps the port unauthenticated (no VLAN assigned).

Cases to test and record, each marked `needs validation` until proven:

| Case | Expected | Result |
| --- | --- | --- |
| MAC never seen by PacketFence | Reject / no access | `needs validation` |
| MAC known to PacketFence but unregistered | Reject / no access | `needs validation` |
| MAC registered with a category other than `UVC` (e.g. `voice`, `gaming`) | Not handled locally; expected to be proxied and rejected | `needs validation` |
| MAC registered as `UVC` but wrong switch port | Accepted (MAC is the identity, not the port) | `needs validation` |
| `UVC` device on a port set to 802.1X rather than MAC-Based | Port does not send a MAB request | `needs validation` |

If a non-`UVC` MAB request is in fact proxied to JumpCloud, note that this sends a MAC address to an external service as a username. See [Security Model](../security/security-model.md).
