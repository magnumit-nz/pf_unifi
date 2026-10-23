# Security Model and Production Hardening

This page states the security properties of the tested design honestly, calls out the weak points, and lists hardening options.

**Nothing on this page has been applied to the tested configuration.** The *Tested* column records what is configured today. *Recommendation* items are advice only and have not been validated in this implementation. The working configuration is not silently rewritten into a "best practice" one.

## Trust model

| Element | Trusted to | Not trusted to |
| --- | --- | --- |
| PacketFence | Make the final authorization decision, assign roles and VLANs | n/a: it is the policy enforcement point |
| JumpCloud RADIUS | Say whether credentials are valid | Decide network access ([Design Decisions D3](../architecture/design-decisions.md#d3-jumpcloud-radius-validates-credentials-only)) |
| JumpCloud LDAP | Supply group membership | Authenticate the user |
| UniFi AP / switch | Enforce the VLAN PacketFence returns; hold the RADIUS shared secret | n/a |
| A MAB device | Nothing: its MAC address is an unauthenticated claim | Identity |

## MAB trust boundary

The actual trust boundary for MAB in this design is the **combination of four things**, not any one of them:

```text
physical port  +  registered MAC  +  UVC category  +  restricted VLAN
```

| Layer | What it contributes | What it does not do |
| --- | --- | --- |
| Physical port | The UniFi port must be set `MAC-Based`; whoever can plug into it can present a MAC | Does not stop someone with physical access |
| Registered MAC | Only MACs an administrator registered can match | A MAC is a claim, not a credential |
| `UVC` category | `node_info.category == "UVC"` selects local handling | Does not authenticate the device; it selects *who handles the request* |
| Restricted VLAN | Limits what a successful (or spoofed) MAB session can reach | Only as strong as the VLAN's firewall policy |

### MAB is spoofable

A MAC address can be read from a device, a label, or the wire and then cloned by an attacker. Anyone who can present a registered `UVC` MAC on a port configured `MAC-Based` is treated as that device. No credential, certificate or challenge is involved. This is inherent to MAB, not a defect of this implementation.

Consequences for the design:

- Treat the `UVC` VLAN as **untrusted-by-default**: least-privilege firewall rules (for example only the camera/NVR traffic needed), no route to staff or management VLANs.
- Physical port security (locked rooms, disabled unused ports, port labelling) is part of the control.
- Keep the `UVC` set small and reconciled ([MAB Device Lifecycle](../operations/mab-device-lifecycle.md)).

**Tested:** MAB is enabled for the `UVC` category as described. **Documented, not changed.**

**Recommendations (not applied):**

- Firewall the `UVC` VLAN to only required destinations and ports.
- Disable unused switch ports; use `MAC-Based` only on ports that need it.
- Alert on a registered `UVC` MAC appearing on a new port, switch, or at a new time (RADIUS audit review; mechanism `needs validation`).
- Where devices support it, move them to 802.1X (certificate-based) and drop MAB.
- Review the effect of UniFi's `MAC Address Filter` and `Port Isolation` on the MAB ports (both disabled as tested); whether either adds protection here `needs validation`.

### UVC role reachable from LDAP rules

PacketFence node categories correspond to roles, and a role assigned by an authentication rule is understood to become the category of an auto-registered node. If any LDAP rule assigns the role `UVC` to a user, that user's auto-registered device would carry category `UVC` and match the MAB filter. This path is **not proven** here (`needs validation`) but is a plausible escalation route.

**Recommendation (not applied):** no authentication rule should assign `UVC`; check this after every rule change.

## SSL Verify Mode = none

**Tested:** the JumpCloud LDAP source uses `Encryption = SSL` on port `636` with `SSL Verify Mode = none`.

**Risk:** the connection is encrypted, but PacketFence does not verify the server certificate. An on-path attacker could present any certificate and intercept the LDAP bind credentials and the group data used for authorization, or return forged group membership (which could grant a higher role). The setting is also in a source that holds the bind DN credentials.

**Documented, not changed.** The existing guidance in this repo is to review it before production.

**Recommendation (not applied):** set certificate verification on, trust the CA chain that signs the JumpCloud LDAP certificate on the PacketFence host, and test. Confirm the connector context trusts the same CA (`needs validation`). Re-run the [LDAP-down drill](../operations/failure-behaviour.md#jumpcloud-ldap-unavailable) afterwards, since a trust failure looks like an outage.

## EAP-MD5 is enabled

**Tested:** `MD5` is listed under *EAP Authentication Types* in both the `default` profile (`GTC, MD5, MSCHAPv2, PEAP, TLS, TTLS`) and the `Jumpcloud-TTLS` profile (`TTLS, MD5, MSCHAPv2, PEAP`).

**Risk:** EAP-MD5 provides no mutual authentication or key derivation and is weak against offline attack of captured exchanges. Leaving it enabled lets a client or attacker request the weakest enabled method; it also keeps `GTC` available in the `default` profile.

**Whether MD5 is required by any client is not established.** The tested clients used PEAP/MSCHAPv2 (see [D8](../architecture/design-decisions.md#d8-jumpcloud-ttls-is-a-profile-name-and-default-not-a-client-mandate)); MSCHAPv2 inner methods remain the working path.

**Documented, not changed.**

**Recommendation (not applied):** remove `MD5` (and `GTC` from `default` if unused) from the EAP profiles, test wireless and wired PEAP/MSCHAPv2 afterwards, and keep `MSCHAPv2` and `PEAP` while clients depend on them. Changes to `tls-common` affect PEAP, TTLS and TLS.

## RADIUS transport

**Tested:** UniFi to PacketFence uses RADIUS over UDP (1812/1813) with a shared secret. UniFi *TLS* is left disabled. Recent FreeRADIUS versions may log a `BlastRADIUS` notice about `require_message_authenticator`; that this system logged it is `needs validation`.

**Notes:**

- The shared secret protects integrity/authentication of the RADIUS exchange; the quality of the secret matters and it must be identical on UniFi and PacketFence.
- The BlastRADIUS notice concerns `Message-Authenticator` handling (which hop it applies to, UniFi-to-PacketFence or the JumpCloud proxy, `needs validation`). It is a log message, not an authentication failure.
- The `Jumpcloud-Radius` source options (`type = auth+acct`) and the proxy hop's transport security to JumpCloud are not further recorded.

**Documented, not changed.**

**Recommendation (not applied):** use long random per-purpose secrets; restrict UDP/1812-1813 to the UniFi device addresses; evaluate setting `require_message_authenticator` once all clients and proxy peers are confirmed to send `Message-Authenticator` (the effect on UniFi and JumpCloud peers `needs validation`); consider RADIUS over TLS only if UniFi and PacketFence both support the combination (`needs validation`).

## Other documented findings

| Finding | Tested value | Concern | Status |
| --- | --- | --- | --- |
| Auto-registration enabled on both 802.1X connection profiles | *Automatically register devices = Enabled* | Any successfully authenticated user device becomes a registered node | Documented, not changed |
| `Max nodes per user = 0` on all roles | `0` | Understood to mean no limit on devices per user (`needs validation`); a user can register unlimited devices | Documented, not changed |
| Single RADIUS home server / LDAP host, no failover recorded | One of each | Availability: see [Failure Behaviour](../operations/failure-behaviour.md) | Documented, not changed |\n| UniFi network-device CoA | `Use CoA = Yes`, `Use Connector For Deauth = Yes` | Configured, but live CoA/disconnect behaviour still needs validation | Documented, not changed |
| Local source-code patch in PacketFence | RADIUS audit controller workaround | May be overwritten on upgrade; unreviewed local code | Documented, not changed; see [Known Issues](../known-issues/radius-audit-timestamp.md) |
| Non-`UVC` MAB requests may be proxied to JumpCloud | Filter only matches `UVC`; others keep `Realm = null` | A MAC address would be sent to an external service as a username (`needs validation`) | Documented, not changed |
| MAB devices share one VLAN | `UVC` VLAN | One compromised/spoofed camera has same reach as all | Documented, not changed |
| `tcpdump` captures during troubleshooting | n/a | Captures contain identities and MACs | Handle as sensitive |
| Role temporary/test objects | Roles list in screenshot includes `default`, `gaming`, and others | Unused roles are unused attack surface; remove temporary test roles before production | Recommendation only |

## Secrets handling

- Never publish RADIUS shared secrets, JumpCloud LDAP bind credentials, API keys or service-account passwords.
- Do not commit secrets, hostnames or directory identifiers; use placeholders (`<RADIUS_SHARED_SECRET>`, `<LDAP_BIND_USER>`, `<JUMPCLOUD_ORG_ID>`).
- Use documentation-only IP and MAC addresses (`192.0.2.0/24`, `00:11:22:33:44:55`) in public material.
- Prefer upstream fixes over permanent local source patches.

## Documentation hygiene

The text uses sanitised examples, but several screenshots were captured from the live lab and show less-sanitised values. They are kept, as required, and the differences are recorded here so they can be reviewed before this repository is made or kept public:

| Screenshot | Observation |
| --- | --- |
| `authsource-jumpcloudradius.png` | The *Host* field shows a literal **public** IP address rather than `<JUMPCLOUD_RADIUS_HOST>`, so the "values replaced" sanitisation notes do not hold for this image. The secret is masked. |
| `roles.png` | Shows real role identifiers and descriptions (e.g. `Executive`, `Wifi-staff-domain`, `BYODY9`..`13`) that differ from the generic names in the text. Some identifiers are blank/redacted. |
| `connprofile-wireless.png` | Shows a literal test SSID (`DS RAD TST`) rather than `<WIRELESS_SSID>` / the text's `CORP-BYOD`. |
| `unifi-packetfence-radius-profile.png` | Shows `192.168.0.20` rather than the text's `192.0.2.10`. Secrets are masked. |\n| `pf_networkdevices-list.png` | Network-device identifiers are redacted, but real-looking descriptions and vendor types remain. |\n| `pfnetworkdevices-switches.png` | Network-device identifier is redacted; tested switch type and default settings remain visible. |\n| `pfnetworkdevices-radius.png` | Shows a live/private CIDR in the window title; the shared secret itself is masked. Consider re-capturing with the identifier redacted. |\n| `pfnetworkdevices-roles.png` | Shows real/test role names and VLAN IDs. Review whether those mappings are acceptable to publish. |

`eap-profile-jumpcloud.png` is unreferenced, duplicates `eapprofile-jumpcloudttls.png` and shows the admin UI header.

**Documented, not changed.** Decide whether to re-capture with placeholders or accept the exposure. Do not edit the screenshots without keeping the originals.

## Production hardening checklist (Recommendations, none applied)

- [ ] Enable certificate verification on the LDAP source and trust the CA
- [ ] Remove `MD5` (and unused `GTC`) from EAP profiles; re-test PEAP/MSCHAPv2
- [ ] Put the `UVC` VLAN behind least-privilege firewall rules
- [ ] Reconcile `UVC` nodes against inventory; define who may register them
- [ ] Ensure no authentication rule assigns `UVC`
- [ ] Disable unused switch ports; restrict `MAC-Based` to required ports
- [ ] Restrict RADIUS ports to UniFi device addresses; use strong unique secrets
- [ ] Evaluate `require_message_authenticator`
- [ ] Decide on a per-user device limit instead of `Max nodes per user = 0`
- [ ] Add a second PacketFence/RADIUS server if availability requires it
- [ ] Run the [failure drills](../operations/failure-behaviour.md) and record results
- [ ] Remove temporary test roles and unused EAP types
- [ ] Replace the local audit-timestamp patch with an upstream fix when available
- [ ] Re-capture or review screenshots for sensitive values
