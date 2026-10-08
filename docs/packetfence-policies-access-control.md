# PacketFence Policies and Access Control

This page is a compact reference for the policy objects used by the documented PacketFence deployment.

> All examples are sanitised. Replace names, groups, VLANs and directory identifiers with values appropriate to your environment.

## Policy Chain

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

For registered MAB devices:

```text
Registered Node
      |
      v
node_info.category
      |
      v
RADIUS Authorize Filter
      |
      v
Local Realm
      |
      v
Device Role / VLAN
```

## Roles

Path:

```text
Configuration
  -> Policies and Access Control
  -> Roles
```

Use explicit roles for different access outcomes, for example:

```text
Staff
Privileged-Staff
Students-9
Students-10
Students-11
Students-12
Students-13
IoT
Guest
Voice
REJECT
```

The tested roles use `Max nodes per user = 0`.

## Realms

Path:

```text
Configuration
  -> Policies and Access Control
  -> Domains
  -> Realms
```

### DEFAULT

```text
EAP = Jumpcloud-TTLS
```

### NULL

```text
RADIUS AUTH = Jumpcloud-Radius
Type = Keyed Balance
Authorize from PacketFence = Enabled
RADIUS ACCT = not configured
Accounting Type = Load Balance
```

### Jumpcloud

```text
RADIUS AUTH = Jumpcloud-Radius
Type = Keyed Balance
Authorize from PacketFence = Enabled
RADIUS ACCT = not configured
Accounting Type = Load Balance
```

## JumpCloud RADIUS Source

Path:

```text
Configuration
  -> Policies and Access Control
  -> Authentication Sources
```

Example:

```text
Name: Jumpcloud-Radius
Host: <JUMPCLOUD_RADIUS_HOST>
Port: 1812
Secret: <RADIUS_SHARED_SECRET>
Timeout: 1
Monitor: Enabled
Use Connector: Enabled
Options: type = auth+acct
```

No PacketFence authentication rules are attached to this source.

## JumpCloud LDAP Source

Example:

```text
Name: Jumpcloud
Host: <JUMPCLOUD_LDAP_HOST>
Port: 636
Encryption: SSL
SSL Verify Mode: none
Dead duration: 60
Connection timeout: 1
Request timeout: 5
Response timeout: 10
Scope: Subtree
Username Attribute: uid
Email Attribute: mail
Cache match: Enabled
Monitor: Enabled
Shuffle: Disabled
Use Connector: Enabled
```

Base DN:

```text
ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com
```

Bind DN:

```text
uid=<LDAP_BIND_USER>,ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com
```

## LDAP Rules

Use `memberOf` to map directory groups to PacketFence roles.

Example:

```text
memberOf equals cn=Staff,ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com

Action:
Role = Staff
```

Use `Matches = All` when every condition in the rule must match.

Order rules from most specific to least specific.

## Connection Profiles

### Wireless

```text
Profile: Jumpcloud-BYOD
Filter: SSID = <WIRELESS_SSID>
Source: Jumpcloud
Automatically register devices: Enabled
Dot1x recompute role from portal: Enabled
MAC Auth recompute role from portal: Disabled
VLAN pool technique: username_hash
```

### Wired

```text
Profile: Jumpcloud_Wired
Filter: Connection Type = Ethernet-EAP
Source: Jumpcloud
Automatically register devices: Enabled
Dot1x recompute role from portal: Enabled
MAC Auth recompute role from portal: Disabled
VLAN pool technique: username_hash
```

## EAP Profiles

### default

```text
Default EAP Type: PEAP
Expires: 60
Types: GTC, MD5, MSCHAPv2, PEAP, TLS, TTLS
TLS Profile: tls-common
TTLS Profile: tls-common
PEAP Profile: tls-common
Fast Profile: default
```

### Jumpcloud-TTLS

```text
Default EAP Type: TTLS
Expires: 60
Types: TTLS, MD5, MSCHAPv2, PEAP
TLS Profile: tls-common
TTLS Profile: tls-common
PEAP Profile: tls-common
Fast Profile: default
```

The `DEFAULT` realm uses `Jumpcloud-TTLS`.

## Local MAB Filter

```ini
[Unifi-MAC-Auth-Local]
scopes=packetfence.authorize
status=enabled
top_op=and
description=Keep UniFi MAC authentication local to PacketFence
merge_answer=yes
condition=node_info.category == "IoT"
answer.0=control:Proxy-To-Realm = local
answer.1=request:Realm = local
```

The two realm assignments are both required in the tested design.

Avoid using `Ethernet-NoEAP` as the sole early authorize-stage detector for MAB because legitimate wired PEAP can be caught too early.

## Dynamic VLAN Reply

PacketFence returns:

```text
Tunnel-Type = VLAN
Tunnel-Medium-Type = IEEE-802
Tunnel-Private-Group-Id = "<VLAN ID>"
```

The VLAN must exist end-to-end on the network.

## See Also

- [Full PacketFence configuration guide](packetfence-configuration.md)
- [UniFi configuration](unifi-configuration.md)
