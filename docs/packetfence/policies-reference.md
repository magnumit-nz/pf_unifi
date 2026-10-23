# PacketFence Policy Reference

A compact map of the policy objects and where each is documented. This page deliberately holds **no configuration values**: the [Configuration Guide](configuration.md) is the single source of truth for settings, and [MAB](mab.md) for the MAB filter.

## Policy chain

```text
Connection Profile
      |
      v
EAP Profile
      |
      v
Realm
      |
      v
JumpCloud RADIUS
      |
      v
JumpCloud LDAP
      |
      v
Authentication Rule
      |
      v
PacketFence Role
      |
      v
Dynamic VLAN
```

This is the logical sequence used for troubleshooting and for explaining the design. It is **not** a verified statement of PacketFence's internal processing order (for example, realm selection and EAP profile selection are tied together by the realm's EAP setting, and the connection profile also drives which source evaluates rules). Treat the order as a reading aid; the internal ordering `needs validation` if you depend on it.

For registered MAB devices:

```text
Registered Node
      |
      v
node_info.category == "UVC"
      |
      v
RADIUS Authorize Filter (Unifi-MAC-Auth-Local)
      |
      v
Local Realm
      |
      v
UVC Role / VLAN
```

## Where each object is documented

| Object | Section |
| --- | --- |
| Roles | [Configuration: Roles](configuration.md#4-roles) |
| Realms (`DEFAULT`, `NULL`, `Jumpcloud`) | [Configuration: Realms](configuration.md#5-realms) |
| JumpCloud RADIUS source | [Configuration: RADIUS source](configuration.md#6-jumpcloud-radius-authentication-source) |
| JumpCloud LDAP source | [Configuration: LDAP source](configuration.md#7-jumpcloud-ldap-authentication-source) |
| LDAP rules and ordering | [Configuration: LDAP rules](configuration.md#8-ldap-authentication-rules), [Worked Example](../architecture/worked-example.md) |
| EAP profiles | [Configuration: EAP profiles](configuration.md#9-eap-profiles) |
| Connection profiles (wireless, wired) | [Configuration: Connection profiles](configuration.md#10-connection-profiles) |
| Network devices / NAS | [Network Devices](network-devices.md) |
| Local MAB filter | [MAB](mab.md) |
| Dynamic VLAN reply | [Configuration: Dynamic VLAN](configuration.md#13-dynamic-vlan-requirements) |

## Rule ordering

Most specific LDAP group rules first (a Recommendation; the actual live order is `to be captured`). Example and reasoning in [Configuration: Rule ordering](configuration.md#8-ldap-authentication-rules).
