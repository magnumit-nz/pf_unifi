# Failure Behaviour

What is expected to happen when each dependency fails. The tested implementation proved the *working* paths; most failure outcomes were **not** exercised. Anything not proven is marked `needs validation`: treat it as a hypothesis derived from the configuration, and fill in the **Observed** column when you test.

Only configuration facts recorded in this repository are quoted as facts.

## Recorded timing and health settings (Tested configuration)

| Source | Setting | Value |
| --- | --- | --- |
| `Jumpcloud-Radius` | Timeout | `1` |
| `Jumpcloud-Radius` | Monitor | Enabled |
| `Jumpcloud-Radius` | Use Connector | Enabled |
| `Jumpcloud` (LDAP) | Dead duration | `60` |
| `Jumpcloud` (LDAP) | Connection / Request / Response timeout | `1` / `5` / `10` |
| `Jumpcloud` (LDAP) | Cache match | Enabled |
| `Jumpcloud` (LDAP) | Monitor | Enabled |
| `Jumpcloud` (LDAP) | Use Connector | Enabled |
| Realms `NULL`, `Jumpcloud` | Type | `Keyed Balance` |
| UniFi RADIUS profile | Interim Update Interval | Disabled |
| Connection profiles | Auto-deregister on accounting stop | Disabled |

There is a single RADIUS home server and a single LDAP host configured in the documented design; no failover target is recorded.

## Summary

| Failure | Who is affected | Expected effect | Status |
| --- | --- | --- | --- |
| JumpCloud RADIUS unavailable | New and re-authenticating 802.1X users (wireless and wired) | Credential check cannot complete: users fail authentication. UVC MAB unaffected | `needs validation` |
| JumpCloud LDAP unavailable | 802.1X users needing a role | Credentials may validate but role cannot be derived: no role/VLAN, or fallback. UVC MAB unaffected (no LDAP dependency recorded) | `needs validation` |
| PacketFence unavailable | Everyone | No RADIUS service: new auths fail; existing sessions persist until re-auth | `needs validation` |
| Connector / DNS failure | 802.1X users via both JumpCloud sources | Sources appear down although the service is up | Partly evidenced (lesson recorded); outcome `needs validation` |
| Accounting failure | Session visibility | Authentication/VLAN placement unaffected; PacketFence session data stale | `needs validation` |
| Missing VLAN on trunk | Any endpoint assigned that VLAN | Auth succeeds, endpoint has no connectivity | Documented; symptoms `needs validation` |
| Missing DHCP | Endpoints in that VLAN | Auth and VLAN correct, no lease | `needs validation` |

## JumpCloud RADIUS unavailable

**Expectation (`needs validation`):** PacketFence cannot obtain Access-Accept from JumpCloud, so proxied (`NULL`/`Jumpcloud` realm) authentications fail. With a 1-second source timeout and *Monitor* enabled, PacketFence is expected to detect and mark the home server as down. Already-connected users keep their VLAN until the session is re-authenticated (UniFi interim updates are off; re-auth interval is not recorded).

**MAB:** UVC devices are handled locally (`Realm = local`) and are not expected to touch JumpCloud RADIUS.

**To test:** block UDP/1812 from PacketFence to JumpCloud (or disable the source) and record: user-facing failure, PacketFence audit entry, time for the monitor to mark it dead, recovery time on restore, UVC MAB result during the outage.

| Observed | |
| --- | --- |
| Result | to be recorded |

## JumpCloud LDAP unavailable

**Expectation (`needs validation`):** the credential check (RADIUS) and the group lookup (LDAP) are separate dependencies. If LDAP is down, a user may authenticate but PacketFence cannot evaluate `memberOf` rules, so no role-derived VLAN can be returned. The outcome could be reject, a default/fallback role, or reuse of a cached match (*Cache match* is enabled). Which of these happens is not recorded.

Dead duration `60` implies the source is skipped for about a minute after failure before being retried (`needs validation`).

**Risk to check:** that a failed lookup never lands a user in a *more* privileged role than intended. This is a security-relevant outcome ([Security Model](../security/security-model.md)).

**To test:** block TCP/636 to JumpCloud LDAP while RADIUS is reachable. Record result for a previously seen and a never-seen user.

## PacketFence unavailable

**Expectation (`needs validation`):** UniFi is configured with one RADIUS server (PacketFence) and no secondary is documented, so no new authentication can succeed. Whether already-connected wired and wireless sessions stay up, and whether UniFi falls back to any local or default network, depends on UniFi behaviour that is not recorded.

**Consider:** a second PacketFence node or UniFi secondary RADIUS server is a design decision not covered by the tested build. See Production Hardening.

**To test:** stop `packetfence-radiusd-auth` (non-production) and record behaviour of an existing session, a new 802.1X login, and a MAB device being plugged in.

## Connector / DNS failure

**Evidence:** both JumpCloud sources have *Use Connector* enabled. A lesson from the build is that PacketFence may use internal/container DNS rather than the host resolver, so the host resolving `ldap.jumpcloud.com` does **not** prove the PacketFence service can. Test name resolution from the PacketFence service context.

**Expectation (`needs validation`):** a connector or DNS fault makes the sources look unreachable and produces the same user-visible effects as the LDAP or RADIUS outages above, while host-level `dig`/`getent` still succeed.

**To test:** break DNS resolution inside the service context (not on the host), then compare host and service-context lookups and note how PacketFence reports it.

## Accounting failure

**Evidence:** accounting goes from UniFi directly to PacketFence on UDP/1813. Realm `RADIUS ACCT` is not configured, so accounting is not proxied to JumpCloud. Interim updates are disabled.

**Expectation (`needs validation`):** accounting loss does not affect Access-Accept or VLAN assignment, which happen at authorization. It degrades PacketFence's view of session start/stop. Because *auto-deregister on accounting stop* is disabled, a missed Stop is not expected to deregister or re-role a node.

**To test:** drop UDP/1813 only; confirm authentication and VLAN still succeed; confirm what disappears from PacketFence.

## Missing VLAN on trunk

**Evidence (documented):** a correct PacketFence Access-Accept cannot compensate for a missing VLAN on the switching path. Every VLAN PacketFence can return must exist, be allowed across required trunks, reach the AP or access switch, have working DHCP, and have the correct gateway/firewall policy.

**Expected symptom (`needs validation`):** RADIUS audit shows success and the correct tunnel attributes; UniFi shows the endpoint authorized, possibly in the assigned VLAN; the endpoint gets no DHCP lease or no connectivity.

**Diagnosis:** check the VLAN exists in UniFi and is tagged on all uplinks between the AP/port and the DHCP/gateway. Do not use Etherlighting as the indicator.

## Missing DHCP

**Expected symptom (`needs validation`):** authentication succeeds and the VLAN is applied, but the endpoint remains without an address (or self-assigns). It is a network-transport fault, separate from authentication, authorization and role selection.

**Diagnosis:** confirm the DHCP scope/relay for the specific VLAN; capture on the VLAN.

## Recording results

Add one dated line per drill here once tested:

| Date | Failure injected | Observed behaviour | Recovery | Doc updated |
| --- | --- | --- | --- | --- |
| to be recorded | | | | |
