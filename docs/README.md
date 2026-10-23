# Documentation Index

Repository Markdown under `docs/` is the canonical documentation. The GitHub Wiki only links here.

## Conventions

| Label | Meaning |
| --- | --- |
| **Tested** | Observed working in the documented implementation; the source of truth |
| **Rationale** | Why the design is shaped this way |
| **Recommendation** | Hardening or process advice, not applied or tested |
| `needs validation` | Expected but unproven |
| `to be captured` | Exists in the live system but not recorded in the repo yet |

`UVC` is the real tested MAB category and role. `IoT` appears only if explicitly labelled as a generic example.

## Reading paths

- **Understand the design:** Architecture Overview -> Design Decisions -> Authentication Matrix -> Worked Example
- **Build it:** PacketFence Configuration -> Network Devices -> MAB -> UniFi Configuration -> Validation Checklist
- **Operate it:** Troubleshooting, MAB Device Lifecycle, Failure Behaviour
- **Harden it:** Security Model, then Known Issues

## Contents

### Architecture
- [Overview](architecture/overview.md): components, flows, status, example values
- [Design Decisions](architecture/design-decisions.md): why PacketFence owns policy and VLANs
- [Authentication Matrix](architecture/authentication-matrix.md): wireless, wired, UVC MAB, unknown MAC
- [Worked Example](architecture/worked-example.md): group -> LDAP rule -> role -> VLAN -> tunnel attributes -> UniFi

### PacketFence
- [Configuration Guide](packetfence/configuration.md): the build path
- [Network Devices](packetfence/network-devices.md): UniFi as NAS (partly `to be captured`)
- [MAB](packetfence/mab.md): `UVC` filter and rejected approaches
- [Policy Reference](packetfence/policies-reference.md): index of policy objects

### UniFi
- [Configuration](unifi/configuration.md): RADIUS profile, dynamic VLANs, MAC-Based ports

### Operations
- [Validation Checklist](operations/validation-checklist.md)
- [Failure Behaviour](operations/failure-behaviour.md)
- [MAB Device Lifecycle](operations/mab-device-lifecycle.md)
- [Troubleshooting](operations/troubleshooting.md)

### Security
- [Security Model and Production Hardening](security/security-model.md)

### Known Issues
- [Index](known-issues/README.md)
- [KI-1 RADIUS audit timestamp bug](known-issues/radius-audit-timestamp.md)

## Screenshots

All screenshots are in [`images/`](images/). Each is used by one page:

| Image | Page |
| --- | --- |
| `roles.png`, `realm-default.png`, `realm-null.png`, `realm-jumpcloud.png`, `authsource-jumpcloudradius.png`, `eapprofile-default.png`, `eapprofile-jumpcloudttls.png`, `connprofile-wireless.png`, `connprofile-wired.png` | [Configuration Guide](packetfence/configuration.md) |
| `radius-filter.png` | [MAB](packetfence/mab.md) |
| `unifi-packetfence-radius-profile.png`, `unifi-mac-based.png` | [UniFi Configuration](unifi/configuration.md) |
| `eap-profile-jumpcloud.png` | Not referenced (earlier capture, retained) |
