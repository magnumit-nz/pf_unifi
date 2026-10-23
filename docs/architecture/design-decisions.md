# Design Decisions

Each decision states what the tested implementation does (**Tested**) and why (**Rationale**). Alternatives that were tried and rejected are in [PacketFence Configuration](../packetfence/configuration.md#mab-approaches-that-failed-or-were-rejected).

## D1. UniFi talks only to PacketFence

**Tested:** UniFi RADIUS profiles point at PacketFence on UDP/1812 (auth) and UDP/1813 (accounting). No UniFi device is configured with JumpCloud as a RADIUS server.

**Rationale:** UniFi can authenticate against JumpCloud RADIUS directly, but then nothing sits in the path that can map identity to a role or return a policy-driven VLAN. Putting PacketFence in the middle gives one enforcement point and one place to log and audit decisions.

## D2. PacketFence owns policy and VLAN assignment

**Tested:** Realms `NULL` and `Jumpcloud` proxy to `Jumpcloud-Radius` with `Authorize from PacketFence = Enabled`. PacketFence evaluates its own authentication rules, selects a role and returns the VLAN tunnel attributes.

**Rationale:**

- **One policy engine.** Role definitions, rule ordering, connection profiles and VLAN mapping live in one system rather than being split between the NAC and the identity provider.
- **JumpCloud RADIUS is used only for its credential answer.** The design does not rely on it for PacketFence roles, per-device-class handling or the VLAN attribute set UniFi consumes here; whether it could supply those was not evaluated.
- **Consistent handling of non-user devices.** MAB devices never reach JumpCloud ([D5](#d5-mab-is-an-explicit-trust-decision-based-on-node-category)); if VLAN policy lived in JumpCloud, user and device paths would need separate policy engines.
- **Replaceability.** The identity provider can change without redefining VLAN policy, because only the credential check and group source are provider-specific.

```text
JumpCloud:    "Are these credentials valid?"
PacketFence:  "What network access should this identity receive?"
```

## D3. JumpCloud RADIUS validates credentials only

**Tested:** The `Jumpcloud-Radius` authentication source has no authentication rules and no administration rules attached. It is referenced only by the realms' `RADIUS AUTH` field.

**Rationale:** Credential validation is the one job the upstream identity provider is authoritative for. Keeping the RADIUS source rule-free prevents accidental policy decisions at the proxy hop and keeps the audit trail clear: JumpCloud answered Accept/Reject; PacketFence decided the outcome.

**Not recorded:** whether JumpCloud RADIUS returns any authorization attributes that PacketFence ignores. `needs validation`.

## D4. JumpCloud LDAP supplies authorization context

**Tested:** The `Jumpcloud` LDAP source (LDAPS/636, Subtree scope, `uid` username attribute) holds the authentication rules. Rules match `memberOf` and assign a PacketFence role. Rules use `Matches = All` and are ordered most specific first. Both connection profiles reference this source.

**Rationale:** Group membership is directory data, not a credential check. Reading it by LDAP lets PacketFence decide the role after JumpCloud RADIUS has accepted the credentials, without asking the identity provider to encode group-to-VLAN policy in RADIUS replies. The group-to-VLAN chain is walked through in the [Worked Example](worked-example.md).

**Trade-off:** the design depends on two JumpCloud services (RADIUS and LDAP) being reachable. See [Failure Behaviour](../operations/failure-behaviour.md).

## D5. MAB is an explicit trust decision based on node category

**Tested:** A RADIUS authorize filter, `Unifi-MAC-Auth-Local`, matches `node_info.category == "UVC"` and sets both `control:Proxy-To-Realm = local` and `request:Realm = local`. Both are needed; setting only `Proxy-To-Realm` was not sufficient.

**Rationale:**

- A MAC address is not a credential, so it must not be sent to a credential-checking upstream (it would not be expected to hold an account for it).
- Matching a resolved PacketFence node category makes "this device may use local MAB" an administrator decision recorded in PacketFence, rather than a guess from username shape or packet type.
- Alternatives (`Ethernet-NoEAP`, MAC-shaped usernames, `Service-Type` alone) were less reliable or did not scale; see the rejected-approaches list.

**Limit of the guarantee:** this controls *which devices are handled locally*; it does not authenticate the device. See [Security Model](../security/security-model.md#mab-trust-boundary).

## D6. Leave the `NULL` realm alone; redirect MAB by filter

**Tested:** `NULL` stays mapped to `Jumpcloud-Radius` and remains part of the known-good PEAP path. MAB is redirected per request by the RADIUS filter.

**Rationale:** Changing `NULL` globally to fix MAB would break working PEAP authentication.

## D7. Separate wired and wireless connection profiles

**Tested:** `Jumpcloud-BYOD` (filter: SSID) and `Jumpcloud_Wired` (filter: `Connection Type = Ethernet-EAP`), both using the `Jumpcloud` source.

**Rationale:** Independent matching, simpler troubleshooting of profile selection, and freedom to change one access method without the other.

## D8. `Jumpcloud-TTLS` is a profile name and default, not a client mandate

**Tested:** The `DEFAULT` realm uses the `Jumpcloud-TTLS` EAP profile, whose *Default EAP Type* is `TTLS` and whose enabled types are `TTLS, MD5, MSCHAPv2, PEAP`. The tested clients authenticated with **PEAP/MSCHAPv2**.

**Rationale / clarification:** "Default EAP Type" governs what the server offers first; it does not prevent a client from negotiating another enabled method. The profile name describes its purpose (the JumpCloud-facing EAP profile), not the method every client used. Removing `PEAP` or `MSCHAPv2` from this profile would break the tested clients.

## D9. Accounting terminates at PacketFence

**Tested:** UniFi sends accounting to PacketFence on UDP/1813. The realms' `RADIUS ACCT` is not configured, so accounting is not proxied by the realm configuration. The `Jumpcloud-Radius` source has `type = auth+acct` in its home-server options.

**Rationale:** PacketFence uses accounting for session correlation. JumpCloud does not need accounting data for the policy design.

**Not recorded:** whether the `type = auth+acct` home-server option has any effect when no realm accounting source is set. `needs validation`.
