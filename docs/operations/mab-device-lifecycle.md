# MAB Device Lifecycle

How a `UVC` device is brought onto the network by MAC Authentication Bypass, checked, replaced and retired. The mechanism (category `UVC`, `Unifi-MAC-Auth-Local` filter, MAC-Based port) is **Tested**; see [MAB](../packetfence/mab.md). The procedural steps below are a **Recommendation** assembled from that mechanism: they were not separately run as a documented runbook, and PacketFence GUI paths for node actions are `to be captured`.

> Because MAB trusts a MAC address ([MAB trust boundary](../security/security-model.md#mab-trust-boundary)), lifecycle discipline is what keeps the trusted set small and accurate.

## Preconditions

- UniFi RADIUS profile points at PacketFence ([UniFi Configuration](../unifi/configuration.md))
- UniFi device defined in PacketFence with the matching secret ([Network Devices](../packetfence/network-devices.md))
- `UVC` role exists and maps to the intended VLAN
- `Unifi-MAC-Auth-Local` filter is enabled

## 1. Registration

1. Record the device's MAC address from the device or its label, not from an observed unknown client, so an attacker-supplied MAC is not registered by mistake.
2. Register the node in PacketFence with `Category = UVC`. Record owner/purpose and physical location in the node notes (field use `to be captured`).
3. Set the target switch port's **Port Security** as tested: `MAC Address Filter` off, `802.1X Control = MAC-Based`, `Idle Timeout = 300`, `Port Isolation` off.
4. Confirm the target VLAN exists on the uplinks and has DHCP.
5. Connect the device.

Do not rely on auto-registration for MAB devices. The 802.1X connection profiles auto-register, but MAB devices are registered deliberately with an explicit category.

## 2. Validation

Run section 5 of the [Validation Checklist](validation-checklist.md#5-uvc-mab):

- RADIUS audit shows the MAC handled locally (not proxied)
- Role `UVC`, expected VLAN attributes returned
- UniFi shows the client in the expected VLAN
- DHCP lease obtained in the UVC subnet
- Node record shows category `UVC`, status registered

Keep the audit entry as evidence.

## 3. Replacement (hardware swap or NIC change)

A replacement presents a new MAC address, so it is a new device from PacketFence's point of view.

1. Register the **new** MAC as `UVC` (steps in Registration).
2. Connect it and validate.
3. Retire the **old** MAC straight away (below); do not leave it registered "just in case".
4. If the new device reuses the same port, no UniFi change is needed.

The reverse also applies: a device that keeps its MAC but moves ports needs no re-registration, because the MAC, not the port, is the identity. If the new port is not set to MAC-Based the device will not authenticate by MAB (`needs validation`). Moving a registered MAC to an unintended port is also a way to gain `UVC` access: review such moves.

## 4. Retirement

1. Unregister or delete the node in PacketFence (or at minimum remove the `UVC` category) so the MAC no longer matches the filter.
2. Disconnect or re-authenticate any active session. Whether PacketFence can force this on UniFi is `needs validation`; otherwise unplug the device or wait for the idle timeout (300 s, `needs validation`).
3. If the port is no longer for MAB, return it to its normal `802.1X Control` setting, or disable it.
4. Confirm the MAC now fails MAB (expected: no access, `needs validation`).
5. Update inventory.

## 5. Periodic review

| Check | Frequency |
| --- | --- |
| List all nodes with `Category = UVC` and reconcile against the physical inventory | Quarterly (suggested) |
| Check no unexpected MACs, especially duplicates or vendor OUIs that do not match cameras | Quarterly |
| Confirm no authentication rule can assign `UVC` to a user | After each rule change |
| List ports set to `MAC-Based` and confirm each is still intended | Quarterly |

## Roles and responsibilities

Not recorded. Decide who may register `UVC` nodes: this is effectively the permission to grant network access without credentials.
