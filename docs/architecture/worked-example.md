# Worked Example: Directory Group to VLAN

This walks one identity through the whole chain: **JumpCloud group -> LDAP rule -> PacketFence role -> VLAN -> RADIUS tunnel attributes -> UniFi**.

> Values below are the sanitised examples used throughout this repository (`Staff`, VLAN `100`, `testuser`). The structure is **Tested**; the specific names and numbers are illustrative. Items not in the repository are marked `to be captured`.

## The chain

```text
JumpCloud user group  "Staff"
        |  member
        v
User  testuser   (authenticated by JumpCloud RADIUS)
        |
        v
PacketFence LDAP source "Jumpcloud"
        |  rule: memberOf equals cn=Staff,ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com
        |  Matches = All
        v
Action: Role = Staff
        |
        v
Role -> VLAN mapping on the UniFi network-device definition (example: Staff -> 100)
        |
        v
Access-Accept
  Tunnel-Type             = VLAN
  Tunnel-Medium-Type      = IEEE-802
  Tunnel-Private-Group-Id = "100"
        |
        v
UniFi AP / switch port places the session in VLAN 100
```

## Step by step

| # | Stage | Where | Value in this example | Status |
| --- | --- | --- | --- | --- |
| 1 | User is a member of a directory group | JumpCloud | Group `Staff` | Tested |
| 2 | Client presents credentials over 802.1X | Client -> UniFi -> PacketFence | PEAP/MSCHAPv2, `User-Name = testuser`, `Realm = "null"` | Tested |
| 3 | PacketFence proxies the credential check | `NULL` realm -> `Jumpcloud-Radius` | JumpCloud RADIUS returns Access-Accept | Tested |
| 4 | PacketFence looks up group membership | LDAP source `Jumpcloud` | Base DN `ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com`; username attribute `uid` | Tested |
| 5 | An authentication rule matches | LDAP source rule list | `memberOf` equals `cn=Staff,ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com`, `Matches = All` | Tested |
| 6 | Rule assigns a role | Rule action | `Role = Staff` (optionally an access duration; one tested rule used 5 days) | Tested |
| 7 | Role is mapped to a VLAN | PacketFence network device -> Roles -> VLAN ID | `Staff -> 100` (illustrative) | Mapping location and mechanism Tested; example value illustrative (see [Network Devices](../packetfence/network-devices.md)) |
| 8 | PacketFence returns tunnel attributes | RADIUS Access-Accept | `Tunnel-Type = VLAN`, `Tunnel-Medium-Type = IEEE-802`, `Tunnel-Private-Group-Id = "100"` | Tested |
| 9 | UniFi applies the VLAN | UniFi RADIUS profile with *RADIUS Assigned VLAN Support* enabled | Session placed in VLAN 100 | Tested |
| 10 | Endpoint gets addressing | VLAN 100 DHCP / gateway | Lease and connectivity | Tested (VLAN must exist on all trunks; see [Failure Behaviour](../operations/failure-behaviour.md#missing-vlan-on-trunk)) |

## Rule ordering matters

Rules are evaluated in order, so a broad rule above a specific one shadows it. Use most-specific-first (**Recommendation**; example order, the live order is `to be captured`):

```text
1. Privileged Staff
2. Specialist Staff
3. Student cohorts
4. Staff BYOD
5. General Wi-Fi Users
```

A user who is in both "Privileged Staff" and "General Wi-Fi Users" must hit the privileged rule first.

## The same shape for MAB, minus JumpCloud

For a `UVC` device none of steps 2 to 6 happen. The identity is the MAC address and the "group" is the PacketFence node category:

```text
Registered node, Category = UVC
   -> Unifi-MAC-Auth-Local filter (Realm = local)
   -> Role UVC
   -> UVC role's VLAN (example 200)
   -> same tunnel attributes, VLAN 200
```

See [MAB](../packetfence/mab.md).

## How to prove each hop

| Hop | Evidence to collect |
| --- | --- |
| 2 | `tcpdump -ni any -s0 'udp port 1812 or udp port 1813'` shows the request from the UniFi device |
| 3 | PacketFence RADIUS audit entry shows the request proxied and accepted |
| 4-6 | PacketFence RADIUS audit / node record shows the role derived from the expected rule |
| 7-8 | Final Access-Accept contains the three tunnel attributes with the expected VLAN |
| 9-10 | UniFi client details show the VLAN; endpoint received a lease in the expected subnet |

Do not use UniFi Etherlighting as proof of the applied VLAN ([Known Issues](../known-issues/README.md)).
