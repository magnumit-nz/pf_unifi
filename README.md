# PacketFence 15 + UniFi + JumpCloud NAC

A practical implementation guide for using **PacketFence 15** as the NAC and policy layer in front of **UniFi** network infrastructure, while retaining **JumpCloud** as the identity and RADIUS authentication source.

> **Sanitisation note**
>
> Hostnames, IP addresses, VLAN IDs, usernames, MAC addresses, directory identifiers and organisational role names in this document have been anonymised or replaced with example values. They do not represent the production environment in which this configuration was tested.

## Status

This design has been tested successfully for:

- Wireless 802.1X using PEAP/MSCHAPv2
- Wired 802.1X using PEAP/MSCHAPv2
- JumpCloud-backed RADIUS authentication
- PacketFence role and VLAN assignment
- UniFi dynamic VLAN enforcement
- MAC Authentication Bypass (MAB) for registered device classes
- PacketFence RADIUS audit logging
- PacketFence 15 audit-log timezone bug identification and workaround

## Architecture

```text
                 +------------------+
                 |    JumpCloud     |
                 |                  |
                 | RADIUS + LDAP    |
                 +---------+--------+
                           |
                 +---------v--------+
                 |   PacketFence    |
                 | NAC / RADIUS     |
                 | Policy / VLAN    |
                 +---------+--------+
                           |
                    RADIUS 1812/1813
                           |
             +-------------+-------------+
             |                           |
       +-----v------+              +-----v------+
       | UniFi APs  |              |UniFi Switch|
       +-----+------+              +-----+------+
             |                           |
        PEAP / 802.1X               802.1X / MAB
             |                           |
       +-----v-----+               +-----v-----+
       |User Device|               | IoT / UVC |
       +-----------+               +-----------+
```

PacketFence is the policy enforcement point.

Normal user authentication is proxied to JumpCloud RADIUS. PacketFence then applies policy and returns the final RADIUS response, including any dynamic VLAN attributes.

Registered infrastructure or IoT devices can use local PacketFence MAB without being sent to JumpCloud.

## Example Network Values

| Purpose | Example |
| --- | --- |
| PacketFence | `192.0.2.10` |
| RADIUS auth | UDP/1812 |
| RADIUS accounting | UDP/1813 |
| Staff VLAN | `100` |
| Student VLAN | `110` |
| IoT / camera VLAN | `200` |
| Management VLAN | `10` |

Use your own addressing and VLAN plan.

# 1. User Authentication Flow

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
  | RADIUS proxy
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
UniFi
  |
  v
Dynamic VLAN
```

A successful final PacketFence reply should contain attributes similar to:

```text
Tunnel-Medium-Type = "IEEE-802"
Tunnel-Private-Group-Id = "100"
Tunnel-Type = "VLAN"
```

# 2. PacketFence RADIUS

UniFi points to PacketFence rather than directly to JumpCloud.

```text
RADIUS server: 192.0.2.10
Authentication: UDP/1812
Accounting: UDP/1813
```

Direct UniFi-to-JumpCloud RADIUS may authenticate users successfully, but PacketFence is what adds the policy layer required for role mapping and dynamic VLAN assignment.

# 3. JumpCloud RADIUS Proxy

Normal user PEAP authentication uses the PacketFence `NULL` realm and is proxied to JumpCloud RADIUS.

```text
Realm = "null"
Service-Type = "Framed-User"
EAP-Type = "PEAP"
User-Name = "testuser"
```

Do not repurpose the `NULL` realm simply to solve MAB. That can break working PEAP authentication.

# 4. JumpCloud LDAP

JumpCloud LDAP is used for authorization and group membership.

```text
cn=Staff,ou=Users,o=<JUMPCLOUD_ORG_ID>,dc=jumpcloud,dc=com
```

Example mappings:

```text
Staff       -> VLAN 100
Students    -> VLAN 110
IoT         -> VLAN 200
```

PacketFence policy rules should be ordered from most specific to least specific because broad rules can shadow specific roles.

# 5. PacketFence Internal DNS and JumpCloud LDAP

A PacketFence deployment may use internal/container DNS services rather than the host resolver directly.

If JumpCloud LDAP appears unreachable, verify name resolution from the relevant PacketFence service environment rather than testing only from the Linux host.

```bash
dig ldap.jumpcloud.com
getent hosts ldap.jumpcloud.com
```

Host DNS resolution working does not necessarily prove PacketFence service/container DNS resolution is working.

# 6. Wireless 802.1X

The tested wireless path was:

```text
macOS / BYOD
  |
UniFi AP
  |
PacketFence
  |
JumpCloud RADIUS
  |
PacketFence policy
  |
Dynamic VLAN
```

PEAP/MSCHAPv2 worked successfully.

# 7. Wired 802.1X

Typical wired requests include:

```text
NAS-Port-Id = "te1/0/14"
NAS-Port-Type = "Ethernet"
Service-Type = "Framed-User"
EAP-Type = "PEAP"
```

A major troubleshooting lesson was that wired PEAP traffic must not be mistaken for MAB at an early PacketFence authorize stage.

A rule such as:

```text
connection_type == "Ethernet-NoEAP"
```

can be unsafe if used too early.

# 8. MAC Authentication Bypass

A recommended approach is:

1. Register the endpoint in PacketFence.
2. Assign it an explicit device category/role.
3. Configure the UniFi switch port for MAC-based authentication.
4. Keep that category local to PacketFence instead of proxying it to JumpCloud.

```text
MAC:      00:11:22:33:44:55
Status:   Registered
Category: IoT
```

```text
IoT -> VLAN 200
```

# 9. MAB RADIUS Filter

The working approach is to identify a trusted local device category and override both the proxy realm and request realm.

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

Both lines are important:

```text
control:Proxy-To-Realm = local
request:Realm = local
```

Setting only `Proxy-To-Realm` was not sufficient in testing.

# 10. Why Match the Assigned Category?

Using:

```text
node_info.category == "IoT"
```

is preferable to guessing from a username or RADIUS packet format.

This means PacketFence administrators explicitly decide which devices can use local MAB.

# 11. MAB Approaches That Did Not Work Well

## `connection_type = Ethernet-NoEAP`

This can be unreliable during early RADIUS authorize processing and may incorrectly catch legitimate wired 802.1X traffic.

## Matching a specific MAC username

Useful as a diagnostic proof, but not scalable.

```text
radius_request.User-Name == "001122334455"
```

## Service-Type matching

`Call-Check` is useful for understanding MAB traffic, but using the resolved PacketFence node/category was cleaner in this implementation.

# 12. UniFi Dynamic VLAN Requirements

UniFi honours standard RADIUS tunnel attributes:

```text
Tunnel-Type = VLAN
Tunnel-Medium-Type = IEEE-802
Tunnel-Private-Group-Id = "<VLAN ID>"
```

Ensure every dynamically assigned VLAN exists end-to-end across trunks, DHCP and gateways.

# 13. UniFi Etherlighting Caveat

Etherlighting did not reliably reflect a VLAN dynamically assigned by RADIUS.

Verify the effective VLAN using PacketFence RADIUS replies, UniFi client details, DHCP and network connectivity.

# 14. BlastRADIUS Messages

Recent FreeRADIUS versions may log:

```text
BlastRADIUS check: Received response to Access-Request with Message-Authenticator.
Please set "require_message_authenticator = true"
```

Treat these separately from actual authentication failures.

# 15. PacketFence RADIUS Audit Timezone Bug

A PacketFence 15 issue was identified where the RADIUS Audit GUI displayed events at the wrong local time.

Example with `Pacific/Auckland` during NZDT:

```text
Actual authentication: 09:03 NZDT
GUI display:           22:03 NZDT
Difference:            +13 hours
```

The timestamp was traced through each layer:

```text
RADIUS Event-Timestamp    correct local time
MariaDB created_at        correct local time
DAL find()                correct local time
DAL search()              correct local time
Unified API               local value incorrectly marked with Z
Browser                   adds local UTC offset
GUI                       wrong by timezone offset
```

Example:

```text
Database:
2026-10-08 09:03:58

Incorrect API value:
2026-10-08T09:03:58Z
```

## Local workaround

Affected controller:

```text
/usr/local/pf/lib/pf/UnifiedApi/Controller/RadiusAuditLogs.pm
```

The workaround interprets the database timestamp using the configured PacketFence timezone and converts it to UTC before API serialization.

After changing the controller, both API services required restart:

```bash
systemctl restart packetfence-pfperl-api
systemctl restart packetfence-api-frontend
```

An upstream fix was prepared so this does not need to remain a site-specific patch.

# 16. Troubleshooting Commands

```bash
systemctl status packetfence-radiusd-auth --no-pager -l
systemctl status packetfence-pfperl-api --no-pager -l
systemctl status packetfence-api-frontend --no-pager -l
timedatectl
date
grep -n '^timezone' /usr/local/pf/conf/pf.conf
tcpdump -ni any -s0 'udp port 1812 or udp port 1813'
grep -n -A8 -B2 '^realm ' /usr/local/pf/raddb/proxy.conf.inc
cat /usr/local/pf/conf/radius_filters.conf
```

# 17. Authentication Troubleshooting Method

```text
1. Did UniFi send the request?
2. Did PacketFence receive it?
3. Which realm did PacketFence select?
4. Was the request local or proxied?
5. Did JumpCloud authenticate it?
6. Did PacketFence resolve the expected group/role?
7. Which policy rule matched?
8. Which VLAN attributes were returned?
9. Did UniFi apply the VLAN?
10. Did the endpoint receive DHCP and network access?
```

Authentication, authorization, role selection, VLAN assignment and network transport are separate stages.

# 18. Working Reference Design

## User 802.1X

```text
User device
  |
UniFi
  |
PacketFence
  |
JumpCloud RADIUS
  |
PacketFence policy
  |
Dynamic user VLAN
```

## Local MAB

```text
Registered device
  |
UniFi MAC authentication
  |
PacketFence
  |
trusted node category
  |
local realm
  |
device role
  |
IoT VLAN
```

# 19. Important Operational Notes

- Treat policy rule ordering as significant.
- Keep normal PEAP authentication on the working JumpCloud proxy realm.
- Keep explicitly registered MAB device classes local.
- Avoid using `Ethernet-NoEAP` alone as an early proxy decision.
- Verify dynamic VLANs end-to-end across trunks and DHCP.
- Do not trust Etherlighting as proof of the effective RADIUS VLAN.
- PacketFence upgrades may overwrite local source-code workarounds.
- Prefer upstream fixes for reproducible PacketFence defects.
- Never publish RADIUS shared secrets, LDAP bind credentials, API tokens or real identity-directory identifiers.

## Disclaimer

This project documents a lab/proof-of-concept implementation and troubleshooting process. Test all configuration changes in a non-production environment and adapt the examples to your own addressing, identity, VLAN and security requirements.
