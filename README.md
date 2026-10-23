# PacketFence 15 + UniFi + JumpCloud NAC

A documented, tested design for using **PacketFence 15** as the NAC and policy layer in front of **UniFi** network infrastructure, with **JumpCloud** providing credential validation (RADIUS) and authorization context (LDAP).

> **Sanitisation note**
>
> Hostnames, IP addresses, VLAN IDs, usernames, MAC addresses, directory identifiers and organisational role names are anonymised or replaced with example values. Some screenshots are less sanitised than the text; see the [Security Model](docs/security/security-model.md#documentation-hygiene).

## At a glance

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

- **PacketFence owns policy and VLAN assignment.** JumpCloud RADIUS only validates credentials; JumpCloud LDAP supplies group membership. See [Design Decisions](docs/architecture/design-decisions.md).
- **Registered `UVC` devices use local MAB** and are handled locally rather than proxied to JumpCloud ([MAB](docs/packetfence/mab.md)).

## Status

**Tested:** wireless and wired 802.1X (PEAP/MSCHAPv2), JumpCloud-backed authentication, role and VLAN assignment, UniFi dynamic VLANs, MAB for registered `UVC` devices, RADIUS audit logging.

**Not proven:** unknown-MAC behaviour and all failure modes. These are marked `needs validation` throughout. Production hardening is documented as recommendations and has **not** been applied; see the [Security Model](docs/security/security-model.md).

## Documentation

| Area | Start here |
| --- | --- |
| Architecture | [Overview](docs/architecture/overview.md) · [Design Decisions](docs/architecture/design-decisions.md) · [Authentication Matrix](docs/architecture/authentication-matrix.md) · [Worked Example](docs/architecture/worked-example.md) |
| PacketFence | [Configuration Guide](docs/packetfence/configuration.md) · [Network Devices](docs/packetfence/network-devices.md) · [MAB](docs/packetfence/mab.md) · [Policy Reference](docs/packetfence/policies-reference.md) |
| UniFi | [Configuration](docs/unifi/configuration.md) |
| Operations | [Validation Checklist](docs/operations/validation-checklist.md) · [Failure Behaviour](docs/operations/failure-behaviour.md) · [MAB Device Lifecycle](docs/operations/mab-device-lifecycle.md) · [Troubleshooting](docs/operations/troubleshooting.md) |
| Security | [Security Model and Hardening](docs/security/security-model.md) |
| Known Issues | [Index](docs/known-issues/README.md) |

The full index, with the labelling conventions (**Tested** / **Rationale** / **Recommendation**, `needs validation`, `to be captured`), is in [docs/README.md](docs/README.md). Repository Markdown is canonical; the [Wiki](https://github.com/magnumit-nz/pf_unifi/wiki) is a navigation layer only.

## Disclaimer

This project documents a lab/proof-of-concept implementation and troubleshooting process. Test all configuration changes in a non-production environment and adapt the examples to your own addressing, identity, VLAN and security requirements.
