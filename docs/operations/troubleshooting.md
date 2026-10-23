# Troubleshooting

Authentication, authorization, role selection, VLAN assignment and network transport are **separate stages**. Find the first stage that fails.

```text
authentication
    != authorization
    != role selection
    != VLAN assignment
    != network transport
```

## Recommended order

```text
1. Did UniFi send the RADIUS request?
2. Did PacketFence receive it?
3. Which connection profile matched?
4. Which EAP profile was selected?
5. Which realm did PacketFence select?
6. Was the request local or proxied?
7. Did JumpCloud RADIUS authenticate the user?
8. Did JumpCloud LDAP return the expected group membership?
9. Which PacketFence authentication rule matched?
10. Which PacketFence role was assigned?
11. Which VLAN attributes were returned?
12. Did UniFi apply the VLAN?
13. Did the endpoint receive DHCP and usable connectivity?
```

This is a troubleshooting sequence, not a verified statement of PacketFence's internal processing order ([Policy Reference](../packetfence/policies-reference.md#policy-chain)).

## Useful commands

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

`tcpdump` is a packet capture: run it deliberately, keep captures short, and treat the output as sensitive (it can contain MAC addresses and usernames; RADIUS secrets are not sent in clear but the traffic is still sensitive).

| Command | Answers |
| --- | --- |
| `systemctl status packetfence-radiusd-auth` | Is the RADIUS auth service running (step 2) |
| `systemctl status packetfence-pfperl-api`, `packetfence-api-frontend` | Are the API services behind the GUI/audit view running |
| `timedatectl`, `date`, `grep '^timezone' pf.conf` | Host and PacketFence time/zone (log correlation) |
| `tcpdump ... udp port 1812 or udp port 1813` | Did UniFi send auth/accounting (step 1-2) |
| `grep ... proxy.conf.inc` | Which realms and home servers PacketFence generated (steps 5-6) |
| `cat radius_filters.conf` | Is the MAB filter present and enabled (step 6) |

## DNS and the connector

PacketFence may use internal/container DNS services rather than the host resolver. If JumpCloud LDAP appears unreachable, test name resolution from the relevant PacketFence service environment, not only the Linux host:

```bash
dig ldap.jumpcloud.com
getent hosts ldap.jumpcloud.com
```

Host resolution working does not prove PacketFence service/container resolution works. Both JumpCloud sources use *Use Connector*.

## Symptom guide

| Symptom | Likely stage | Check |
| --- | --- | --- |
| No request at PacketFence | 1-2 | UniFi profile address/port/secret; device defined in PacketFence ([Network Devices](../packetfence/network-devices.md)); firewall |
| Wired PEAP treated as MAB / proxied wrongly | 5-6 | Make sure no filter uses `Ethernet-NoEAP` as an early detector |
| MAB device proxied to JumpCloud | 6 | Node category is exactly `UVC`; **both** filter answers present; filter enabled |
| User authenticates but gets wrong role | 8-10 | Rule ordering (specific before broad); group DN; `Matches = All` |
| Right role, wrong or no VLAN | 11-12 | Role-to-VLAN mapping; reply contains all three tunnel attributes |
| Right VLAN attributes, no connectivity | 12-13 | VLAN exists on trunks, DHCP scope, gateway/firewall; not Etherlighting |
| LDAP looks down, host DNS fine | connector | Resolve from the service context |
| GUI audit times 13 h off | display only | [Known Issues](../known-issues/radius-audit-timestamp.md) |
| `BlastRADIUS check ... require_message_authenticator` in logs | log noise | See below |

## BlastRADIUS log messages

Recent FreeRADIUS versions may log:

```text
BlastRADIUS check: Received response to Access-Request with Message-Authenticator.
Please set "require_message_authenticator = true"
```

Treat these separately from actual authentication failures. They are a hardening notice, not the cause of a failed login. See [Security Model](../security/security-model.md#radius-transport) and [Known Issues](../known-issues/README.md).

## Related

- [Validation Checklist](validation-checklist.md)
- [Failure Behaviour](failure-behaviour.md)
- [Known Issues](../known-issues/README.md)
