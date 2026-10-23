# Architecture Overview

This page describes what was built, how a request moves through it, and how to read the rest of this documentation set.

> **Sanitisation note**
>
> Hostnames, IP addresses, VLAN IDs, usernames, MAC addresses, directory identifiers and organisational role names are anonymised or replaced with example values. They do not represent the production environment in which this configuration was tested. See [Security Model](../security/security-model.md#documentation-hygiene) for screenshot caveats.

## How to read this documentation

Every statement in this set belongs to one of three categories. They are labelled wherever they could be confused.

| Label | Meaning |
| --- | --- |
| **Tested** | Configuration or behaviour observed working in the documented implementation. This is the source of truth. |
| **Rationale** | Why the tested design is shaped the way it is. Explains intent; does not add configuration. |
| **Recommendation** | Production hardening or process advice that was **not** applied or tested here. Never presented as the working configuration. |

Two further markers flag gaps:

| Marker | Meaning |
| --- | --- |
| `needs validation` | Behaviour is expected or hypothesised but has not been proven in this implementation. Test before relying on it. |
| `to be captured` | A setting or screenshot that exists in the live system but is not recorded in this repository yet. |

## Status

**Tested** in the documented implementation:

- Wireless 802.1X using PEAP/MSCHAPv2
- Wired 802.1X using PEAP/MSCHAPv2
- JumpCloud-backed RADIUS authentication
- PacketFence role and VLAN assignment
- UniFi dynamic VLAN enforcement
- MAC Authentication Bypass (MAB) for registered `UVC` devices
- PacketFence RADIUS audit logging
- PacketFence 15 audit-log timezone bug identification and workaround ([Known Issues](../known-issues/README.md))

Not tested or not recorded: see the Authentication Matrix ([unknown MAC behaviour](authentication-matrix.md#unknown-and-unregistered-mac-behaviour)) and [Failure Behaviour](../operations/failure-behaviour.md).

## Components

| Component | Role in this design |
| --- | --- |
| UniFi APs and switches | RADIUS clients (NAS). Send 802.1X and MAC-based requests to PacketFence and enforce the returned VLAN. |
| PacketFence 15 | NAC and policy engine. Owns realm selection, authorization, role assignment and VLAN assignment. Terminates all RADIUS from UniFi. |
| JumpCloud RADIUS | Credential validation only. Reached by PacketFence as a proxy target. |
| JumpCloud LDAP | Authorization context (group membership). Queried by PacketFence. |

## Reference architecture

```text
Client
  |
  v
UniFi AP / Switch
  |
  | RADIUS 1812 / 1813
  v
PacketFence
  |
  +---- Authentication ----> JumpCloud RADIUS
  |
  +---- Authorization -----> JumpCloud LDAP
  |
  v
PacketFence role / policy
  |
  v
Dynamic VLAN
```

For local MAB:

```text
UVC / registered device
  |
  v
UniFi switch port configured MAC-Based
  |
  v
PacketFence
  |
  | node_info.category == "UVC"
  v
Unifi-MAC-Auth-Local RADIUS filter
  |
  +-- control:Proxy-To-Realm = local
  +-- request:Realm = local
  |
  v
PacketFence local policy
  |
  v
UVC role / assigned VLAN
```

UniFi never talks to JumpCloud directly. Direct UniFi-to-JumpCloud RADIUS may authenticate users, but it lacks the PacketFence policy layer needed for role mapping and dynamic VLAN assignment.

## Example network values

| Purpose | Example |
| --- | --- |
| PacketFence | `192.0.2.10` |
| RADIUS auth | UDP/1812 |
| RADIUS accounting | UDP/1813 |
| Staff VLAN | `100` |
| Student VLAN | `110` |
| UVC (camera) VLAN | `200` |
| Management VLAN | `10` |
| Test username | `testuser` |
| Test MAC | `00:11:22:33:44:55` |
| Timezone | `Pacific/Auckland` |

Use your own addressing and VLAN plan.

## Authentication flow for users (PEAP/MSCHAPv2)

```text
Client
  |
  | PEAP / MSCHAPv2
  v
UniFi AP or Switch
  |
  | RADIUS
  v
PacketFence
  |
  | RADIUS proxy (NULL realm -> Jumpcloud-Radius)
  v
JumpCloud RADIUS
  |
  | Access-Accept
  v
PacketFence
  |
  | LDAP group lookup / policy
  | Role selection
  v
Access-Accept
Tunnel-Type = VLAN
Tunnel-Medium-Type = IEEE-802
Tunnel-Private-Group-Id = <VLAN>
  |
  v
UniFi  -->  Dynamic VLAN
```

Request attributes seen on a normal proxied user authentication:

```text
Realm = "null"
Service-Type = "Framed-User"
EAP-Type = "PEAP"
User-Name = "testuser"
```

A successful final PacketFence reply carries attributes similar to:

```text
Tunnel-Medium-Type = "IEEE-802"
Tunnel-Private-Group-Id = "100"
Tunnel-Type = "VLAN"
```

Do not repurpose the `NULL` realm to solve MAB; that can break working PEAP authentication. MAB is redirected selectively by a RADIUS filter instead ([MAB](../packetfence/mab.md)).

## Operational rules carried by the design

These are **Tested** behaviours or lessons, collected from the build:

- Policy rule ordering is significant; most specific first.
- Keep normal PEAP authentication on the working JumpCloud proxy realm (`NULL`).
- Keep explicitly registered MAB device classes local.
- Do not use `Ethernet-NoEAP` alone as an early proxy decision.
- Verify dynamic VLANs end-to-end across trunks, DHCP and gateway.
- Do not treat UniFi Etherlighting as proof of the effective RADIUS VLAN ([Known Issues](../known-issues/README.md)).
- PacketFence upgrades may overwrite local source-code workarounds; prefer upstream fixes.
- Never publish RADIUS shared secrets, LDAP bind credentials, API tokens or real directory identifiers.

## Document map

| Area | Pages |
| --- | --- |
| Architecture | this page, [Design Decisions](design-decisions.md), [Authentication Matrix](authentication-matrix.md), [Worked Example](worked-example.md) |
| PacketFence | [Configuration](../packetfence/configuration.md), [Network Devices](../packetfence/network-devices.md), [MAB](../packetfence/mab.md), [Policy Reference](../packetfence/policies-reference.md) |
| UniFi | [Configuration](../unifi/configuration.md) |
| Operations | [Validation Checklist](../operations/validation-checklist.md), [Failure Behaviour](../operations/failure-behaviour.md), [MAB Device Lifecycle](../operations/mab-device-lifecycle.md), [Troubleshooting](../operations/troubleshooting.md) |
| Security | [Security Model and Hardening](../security/security-model.md) |
| Known Issues | [Index](../known-issues/README.md), [RADIUS audit timestamp bug](../known-issues/radius-audit-timestamp.md) |

## Disclaimer

This project documents a lab/proof-of-concept implementation and troubleshooting process. Test all configuration changes in a non-production environment and adapt the examples to your own addressing, identity, VLAN and security requirements.
