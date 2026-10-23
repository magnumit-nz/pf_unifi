# Known Issues

Issues and pitfalls observed while building and testing the design. Each entry says what was seen, the impact, and the workaround or decision. The main build path links here instead of repeating the detail.

| ID | Issue | Impact | Status |
| --- | --- | --- | --- |
| KI-1 | [PacketFence 15 RADIUS audit timestamp shown wrong](radius-audit-timestamp.md) | GUI shows event times offset by the local UTC offset (+13 h in the observed case) | Local workaround applied; upstream fix prepared (link `to be captured`) |
| KI-2 | UniFi Etherlighting does not reliably show a RADIUS-assigned VLAN | Misleading visual indicator | Verify with RADIUS replies, UniFi client details, DHCP |
| KI-3 | `Ethernet-NoEAP` as an early authorize-stage MAB detector | Catches legitimate wired PEAP before the inner EAP exchange completes | Not used; MAB uses `node_info.category == "UVC"` |
| KI-4 | Setting only `control:Proxy-To-Realm = local` for MAB | Request retains `Realm = null` and may still follow the JumpCloud proxy path | Both `Proxy-To-Realm` and `Realm` are set |
| KI-5 | Host DNS working does not prove PacketFence container/connector DNS | LDAP can look unreachable while host `dig`/`getent` succeed | Test from the PacketFence service context |
| KI-6 | FreeRADIUS `BlastRADIUS` / `require_message_authenticator` log messages | Log noise; not an authentication failure | Documented; see [Security Model](../security/security-model.md#radius-transport) |
| KI-7 | Local PacketFence source-code workarounds can be overwritten by upgrades | Bug returns after upgrade | Re-apply or replace with upstream fix |

## KI-2: Etherlighting

Etherlighting did not reliably reflect a VLAN dynamically assigned by RADIUS. Verify the effective VLAN from the PacketFence RADIUS reply, UniFi client details, DHCP and connectivity.

## KI-3 and KI-4: MAB pitfalls

See [MAB](../packetfence/mab.md#mab-approaches-that-failed-or-were-rejected) and the [filter](../packetfence/mab.md#radius-authorize-filter-tested).

## KI-5: DNS and the connector

See [Troubleshooting](../operations/troubleshooting.md#dns-and-the-connector).

## KI-6: BlastRADIUS

See [Troubleshooting](../operations/troubleshooting.md#blastradius-log-messages).

## KI-7: Upgrades and local patches

PacketFence upgrades may overwrite local source-code workarounds. Keep a record of every local patch (file, reason, date) and re-test after upgrades. Prefer upstream fixes for reproducible defects.
