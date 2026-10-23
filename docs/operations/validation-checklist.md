# Validation Checklist

A practical post-build and post-change test list. Tick each item and record the evidence. Items that go beyond what the tested implementation proved are marked `needs validation`: running them is how they stop being unproven.

Replace example values (`192.0.2.10`, VLAN `100`/`200`, `testuser`, `00:11:22:33:44:55`) with your own.

## 1. Platform

- [ ] PacketFence timezone is a named IANA zone: `grep -n '^timezone' /usr/local/pf/conf/pf.conf`
- [ ] Host time and zone agree: `timedatectl`, `date`
- [ ] RADIUS service is active: `systemctl status packetfence-radiusd-auth --no-pager -l`
- [ ] API services are active: `systemctl status packetfence-pfperl-api --no-pager -l` and `systemctl status packetfence-api-frontend --no-pager -l`
- [ ] Generated realms are as expected: `grep -n -A8 -B2 '^realm ' /usr/local/pf/raddb/proxy.conf.inc`
- [ ] MAB filter is present and enabled: `cat /usr/local/pf/conf/radius_filters.conf` shows `[Unifi-MAC-Auth-Local]` with `status=enabled`

## 2. Connectivity and dependencies

- [ ] From UniFi, PacketFence is reachable on UDP/1812 and UDP/1813 (`tcpdump -ni any -s0 'udp port 1812 or udp port 1813'` on PacketFence shows traffic during a test)
- [ ] Every UniFi AP and switch is defined in PacketFence with the matching secret ([Network Devices](../packetfence/network-devices.md))
- [ ] JumpCloud LDAP name resolves from the PacketFence service/connector context, not only the host: `dig ldap.jumpcloud.com`, `getent hosts ldap.jumpcloud.com`
- [ ] JumpCloud RADIUS and LDAP sources show healthy in PacketFence (both have *Monitor* enabled)

## 3. Wireless 802.1X (PEAP/MSCHAPv2)

- [ ] Test user in each directory group connects to the SSID matched by `Jumpcloud-BYOD`
- [ ] RADIUS audit shows `Realm = null`, `EAP-Type = PEAP`, request proxied to JumpCloud and accepted
- [ ] PacketFence derives the **expected role** for each group (check specific groups before broad ones)
- [ ] Final Access-Accept carries `Tunnel-Type = VLAN`, `Tunnel-Medium-Type = IEEE-802`, `Tunnel-Private-Group-Id = "<expected VLAN>"`
- [ ] UniFi client details show the expected VLAN
- [ ] Endpoint receives a lease in the expected subnet and reaches the expected resources
- [ ] User with a wrong password is rejected

## 4. Wired 802.1X (PEAP/MSCHAPv2)

- [ ] Port is configured for 802.1X, not MAC-Based
- [ ] RADIUS audit shows `NAS-Port-Type = "Ethernet"`, `EAP-Type = "PEAP"`, connection profile `Jumpcloud_Wired`
- [ ] Request is **not** treated as MAB (not caught by `Ethernet-NoEAP`)
- [ ] Role, VLAN attributes, UniFi client VLAN and DHCP all as expected, as for wireless

## 5. UVC MAB

- [ ] Device is registered in PacketFence with `Category = UVC` ([lifecycle](mab-device-lifecycle.md))
- [ ] Port has `802.1X Control = MAC-Based`, `MAC Address Filter` disabled, `Idle Timeout = 300`, `Port Isolation` disabled
- [ ] RADIUS audit shows the MAC-shaped username handled locally (filter matched; **not** sent to JumpCloud)
- [ ] Role `UVC`, VLAN attributes for the UVC VLAN returned, port placed in that VLAN, DHCP lease obtained
- [ ] Both filter answers are present: `control:Proxy-To-Realm = local` **and** `request:Realm = local`
- [ ] Moving a `UVC` device to a port set to 802.1X does not silently grant access `needs validation`

## 6. Negative tests

- [ ] Unregistered MAC on a MAB port gets no access `needs validation` ([matrix](../architecture/authentication-matrix.md#unknown-and-unregistered-mac-behaviour))
- [ ] MAC registered under a non-`UVC` category on a MAB port is not handled locally `needs validation`
- [ ] Unknown username / user not in any rule's group gets the intended outcome (reject or default role) `needs validation`
- [ ] Wrong RADIUS shared secret on a test device is refused and logged

## 7. Rule ordering

- [ ] A user in both a specific and a broad group gets the specific role
- [ ] No LDAP rule can assign the `UVC` role to a user (see [Security Model](../security/security-model.md#uvc-role-reachable-from-ldap-rules))

## 8. Accounting

- [ ] Accounting Start appears in PacketFence for a new session
- [ ] Accounting Stop appears on disconnect (note: *Automatically deregister devices on accounting stop* is disabled in the tested profiles, so a Stop must not deregister the node)
- [ ] Interim updates are off as tested; confirm session visibility is acceptable for your needs

## 9. Network path

- [ ] Every VLAN PacketFence can return exists and is tagged on all relevant trunks/uplinks
- [ ] DHCP scope and gateway/firewall policy exist for each such VLAN
- [ ] Verified by connectivity and DHCP, **not** Etherlighting

## 10. Failure drills (non-production first)

Run the drills in [Failure Behaviour](failure-behaviour.md) and record the observed outcomes there. Each remains `needs validation` until you do.

## Evidence to keep

For each test: timestamp, test identity (user or MAC), switch/AP and port or SSID, RADIUS audit entry, role, VLAN attribute, UniFi client VLAN, DHCP lease. Never record secrets. When auditing times, remember the [audit timestamp bug](../known-issues/radius-audit-timestamp.md).
