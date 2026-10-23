# KI-1: RADIUS Audit Timestamp Bug (PacketFence 15)

The RADIUS Audit GUI displays events at the wrong local time. This does not affect authentication, roles or VLANs; it affects how audit times are displayed, which matters when correlating logs.

## Summary

PacketFence 15 was observed serializing a local `radius_audit_log.created_at` value with a UTC `Z` suffix without converting it first. The browser then applies the local offset again.

Example with `Pacific/Auckland` during NZDT:

```text
Actual authentication: 09:03 NZDT
GUI display:           22:03 NZDT
Difference:            +13 hours
```

Example values:

```text
Database:
2026-10-08 09:03:58

Incorrect API value:
2026-10-08T09:03:58Z
```

## Evidence: where the value goes wrong

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

## Affected component

```text
/usr/local/pf/lib/pf/UnifiedApi/Controller/RadiusAuditLogs.pm
```

## Local workaround (applied)

The workaround interprets the database timestamp using the configured PacketFence timezone and converts it to UTC before API serialization. The patch text itself is not stored in this repository (`to be captured`).

After changing the controller, **both** API services must be restarted:

```bash
systemctl restart packetfence-pfperl-api
systemctl restart packetfence-api-frontend
```

## What not to do

Do not work around this by changing the server or PacketFence timezone. The timezone is correct; the serialization is the defect.

## Upstream

An upstream fix was prepared for the PacketFence project so this does not need to remain a site-specific patch. The upstream issue/PR reference is not recorded here (`to be captured`).

## Maintenance

- A PacketFence upgrade may overwrite the patched controller; re-check the audit display after upgrades.
- Remove the local patch once an upstream release contains the fix.
- Verify after any change using the checks in [Troubleshooting](../operations/troubleshooting.md) (`timedatectl`, `date`, `grep '^timezone' /usr/local/pf/conf/pf.conf`).
