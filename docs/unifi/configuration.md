# UniFi Configuration for PacketFence

This page documents the UniFi-side configuration used to send 802.1X and MAC-based authentication requests to PacketFence.

> **Public documentation notice**
>
> IP addresses, VLAN IDs and other environment-specific values shown in this guide have been replaced with documentation examples. RADIUS shared secrets must never be published.
>
> This page records the **Tested** UniFi settings. Design rationale is in [Design Decisions](../architecture/design-decisions.md).

---

## Create the PacketFence RADIUS Profile

In UniFi Network, create a RADIUS profile for PacketFence.

Example profile name:

```text
packetfence
```

> The screenshot was taken from a lab profile named `pf` with a different example address (`192.168.0.20`) from the `192.0.2.10` used in the text. Use your own PacketFence address; the shared secret is never shown.

![UniFi PacketFence RADIUS profile](../images/unifi-packetfence-radius-profile.png)

### RADIUS Assigned VLAN Support

Enable RADIUS-assigned VLAN support for both:

- **Wired Networks**
- **Wireless Networks**

This allows PacketFence to return standard RADIUS tunnel attributes such as:

```text
Tunnel-Type = VLAN
Tunnel-Medium-Type = IEEE-802
Tunnel-Private-Group-Id = "<VLAN ID>"
```

and have UniFi apply the returned VLAN to the authenticated endpoint.

### TLS

Leave **TLS disabled** when using standard RADIUS over UDP.

The tested configuration uses:

```text
Authentication: UDP/1812
Accounting:     UDP/1813
```

### Authentication Server

Add the PacketFence server as the authentication server.

Example:

| Setting | Value |
| --- | --- |
| IP Address | `192.0.2.10` |
| Port | `1812` |
| Shared Secret | `<RADIUS_SHARED_SECRET>` |

The shared secret must exactly match the secret configured for the corresponding UniFi NAS/network device in PacketFence ([Network Devices](../packetfence/network-devices.md)).

Do not commit the real secret to source control.

### Accounting Server

Enable **Accounting Servers** and configure the same PacketFence server.

Example:

| Setting | Value |
| --- | --- |
| IP Address | `192.0.2.10` |
| Port | `1813` |
| Shared Secret | `<RADIUS_SHARED_SECRET>` |

RADIUS accounting provides PacketFence with session information after authentication.

### Interim Update Interval

The tested configuration leaves **Interim Update Interval disabled**.

This is not required for the basic authentication and dynamic VLAN workflow.

### Resulting RADIUS Path

```text
UniFi AP / Switch
       |
       +---- UDP/1812 ----> PacketFence authentication
       |
       +---- UDP/1813 ----> PacketFence accounting
```

PacketFence then proxies normal user authentication to the configured identity provider and returns the final role/VLAN information to UniFi.


## MAC Authentication Bypass on Switch Ports

For devices that cannot perform normal 802.1X authentication, configure the applicable UniFi switch port for MAC-based authentication.

Open the switch port's **Port Security** settings and configure:

| Setting | Value |
| --- | --- |
| MAC Address Filter | Disabled |
| 802.1X Control | `MAC-Based` |
| Idle Timeout | `300 seconds` |
| Port Isolation | Disabled |

![UniFi MAC-based port authentication](../images/unifi-mac-based.png)

With `802.1X Control` set to `MAC-Based`, UniFi uses the connected device's MAC address as the identity presented to RADIUS.

The authentication flow is:

```text
Device
  |
  | MAC address
  v
UniFi switch port
  |
  | MAC-based RADIUS authentication
  v
PacketFence
  |
  | node_info.category == "UVC"
  v
Unifi-MAC-Auth-Local filter
  |
  +-- Proxy-To-Realm = local
  +-- Realm = local
  |
  v
PacketFence local policy
  |
  v
UVC role / assigned VLAN
```

The endpoint must already be registered in PacketFence and assigned to the expected category:

```text
Category = UVC
```

PacketFence then matches the local MAB authorize filter and prevents the request from being proxied to JumpCloud RADIUS. See [MAB](../packetfence/mab.md).

Normal wired 802.1X clients should continue to use the `Ethernet-EAP` connection profile rather than MAC-based authentication.

### Port-level notes

- **Tested** settings are exactly those in the table above. `MAC Address Filter` and `Port Isolation` were left disabled.
- The 300-second idle timeout is as tested. How it interacts with silent devices (camera idle, re-authentication after timeout) is `needs validation`.
- Port-level MAB is only as strong as physical access to the port. See the [MAB trust boundary](../security/security-model.md#mab-trust-boundary).
- Every VLAN PacketFence can return must exist and be tagged on the switch uplinks/trunks. See [Failure Behaviour](../operations/failure-behaviour.md#missing-vlan-on-trunk).

## UniFi Etherlighting Caveat

Etherlighting did not reliably reflect a VLAN dynamically assigned by RADIUS. Verify the effective VLAN from PacketFence RADIUS replies, UniFi client details, DHCP and connectivity. See [Known Issues](../known-issues/README.md).

## Related

- [PacketFence Network Devices](../packetfence/network-devices.md)
- [Authentication Matrix](../architecture/authentication-matrix.md)
- [Validation Checklist](../operations/validation-checklist.md)
