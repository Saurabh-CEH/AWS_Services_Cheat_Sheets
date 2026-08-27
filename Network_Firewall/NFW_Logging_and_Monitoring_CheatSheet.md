# AWS Network Firewall - Logging & Monitoring Cheat Sheet

## Overview

Network Firewall visibility comes from **firewall logs** (ALERT, FLOW, and TLS log types) delivered to CloudWatch Logs / S3 / Firehose, and **CloudWatch metrics** for the firewall. Logs are essential for tuning rules (especially before switching from `alert` to `drop`).

**Key point:** You choose which **log types** to enable — **ALERT** (rule matches), **FLOW** (connection metadata sent to the stateful engine), and **TLS** (when TLS inspection is on). Passing traffic that never matches a rule mostly shows up in FLOW, not ALERT.

---

## Log Types

| Log type | Contents                                                        |
| -------- | --------------------------------------------------------------- |
| **ALERT**| Traffic matching stateful rules with `alert` or `drop` action   |
| **FLOW** | Connection/flow metadata for traffic forwarded to the stateful engine |
| **TLS**  | TLS inspection details (only when TLS inspection is configured) |

Destinations: **CloudWatch Logs**, **Amazon S3**, **Amazon Data Firehose** (per log type; you can send different types to different destinations).

---

## ALERT vs FLOW

| Question                                   | Look in |
| ------------------------------------------ | ------- |
| Which rule blocked/alerted this traffic?   | ALERT   |
| What connections passed through / were forwarded? | FLOW |
| Why did stateful inspection not see it?    | Neither (stateless `pass` bypasses SFE — not logged as FLOW) |

> If traffic is `aws:pass`ed at the stateless layer, it never reaches the stateful engine and won't appear in FLOW logs. Forward to SFE to get FLOW visibility.

---

## CloudWatch Metrics (namespace `AWS/NetworkFirewall`)

| Metric                        | Meaning                                            |
| ----------------------------- | -------------------------------------------------- |
| `Packets` / `ReceivedPackets` | Packet counts processed                            |
| `DroppedPackets`              | Packets dropped by rules                           |
| `PassedPackets`               | Packets passed                                     |
| `InvalidDroppedPackets` / `OtherDroppedPackets` | Drop categorization             |
| `TLSErrors` (with TLS inspection) | TLS handshake/inspection errors                |

> Custom stateless actions can emit **named CloudWatch dimensions**, letting you count matches for specific rule categories.

---

## CLI

```bash
# Enable ALERT + FLOW logging (ALERT to S3, FLOW to CloudWatch)
aws network-firewall update-logging-configuration \
  --firewall-name my-fw \
  --logging-configuration '{
    "LogDestinationConfigs": [
      {"LogType":"ALERT","LogDestinationType":"S3","LogDestination":{"bucketName":"aws-nfw-logs","prefix":"alert/"}},
      {"LogType":"FLOW","LogDestinationType":"CloudWatchLogs","LogDestination":{"logGroup":"/nfw/flow"}}
    ]
  }'

# Add TLS logs (when TLS inspection is configured)
# ... append {"LogType":"TLS","LogDestinationType":"S3","LogDestination":{"bucketName":"aws-nfw-logs","prefix":"tls/"}}
```

---

## Athena / Insights examples

```
# CloudWatch Logs Insights — top blocked SNIs from ALERT logs
fields event.tls.sni, event.alert.action
| filter event.alert.action = "blocked"
| stats count(*) as hits by event.tls.sni
| sort hits desc | limit 20
```

---

## Pricing

| Item                     | Cost                                          |
| ------------------------ | --------------------------------------------- |
| Firewall logs            | Vended-logs delivery + destination storage    |
| CloudWatch metrics       | Standard CloudWatch charges                    |

> Log cost scales with traffic volume and how many log types you enable. FLOW on a busy firewall is high-volume — prefer S3 for FLOW, and use ALERT/TLS selectively.

---

## Gotchas & Caveats

1. **ALERT only shows `alert`/`drop` matches** — traffic that simply passes won't appear there; use FLOW to see forwarded connections.
2. **Stateless `pass` traffic isn't in FLOW** — it bypasses the stateful engine; forward to SFE if you want it logged.
3. **You must enable each log type explicitly** — no logs by default; TLS logs only exist when TLS inspection is configured.
4. **Destination permissions required** — S3 bucket policy / CloudWatch role / Firehose policy, or delivery silently fails.
5. **FLOW volume is large** — on high-throughput firewalls, FLOW logs can be expensive; route them to S3 (+ Parquet/Athena).
6. **Log delivery isn't instant** — expect minutes of delay; not a live packet capture.
7. **Log schema is nested JSON (Suricata EVE-style)** — Athena/Insights queries must match the JSON structure.
8. **Different log types can go to different destinations** — but each needs its own destination config and permissions.
9. **Metrics vs logs** — metrics tell you *how much* dropped; logs tell you *what/why*. Use both.
10. **TLS errors surface as metrics/logs** — a spike in `TLSErrors` often means cert/scope misconfiguration, not attacks.
11. **Changing logging config** applies going forward; it doesn't backfill past traffic.
12. **Managed rule alerts** appear in ALERT logs like any rule — validate managed groups in alert mode using these logs.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| No logs delivered                           | Log type not enabled / destination perms        | Enable log type; fix bucket/role/stream permissions        |
| Passing traffic not visible                 | Stateless `pass` bypasses SFE                    | Forward to SFE for FLOW visibility                         |
| Can't tell which rule dropped               | Only FLOW enabled                                | Enable ALERT logs                                          |
| High log cost                               | FLOW on busy firewall to CloudWatch              | Send FLOW to S3 (Parquet); scope log types                 |
| Athena parse errors                         | Query doesn't match nested EVE JSON              | Build schema to match the JSON structure                   |
| Spike in TLSErrors                          | Cert/scope misconfig                             | Review TLS inspection config and certificates              |

---

## Best Practices

1. **Enable ALERT + FLOW** during rollout; add TLS logs when TLS inspection is on.
2. **Send high-volume FLOW to S3** (Parquet) for cheap querying; ALERT to S3/CloudWatch for fast triage.
3. **Roll out rules in `alert` mode**, watch ALERT logs, then switch to `drop`.
4. **Alarm on `DroppedPackets` and `TLSErrors`** to catch misconfig/attacks.
5. **Forward to SFE** for the traffic you want in FLOW logs.
6. **Centralize logs** cross-account for security teams.
7. **Build standard Athena queries** (top blocked SNIs, top talkers) as a runbook.

---

## Useful Links

- [Firewall logging](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-logging.html)
- [Log types & destinations](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-logging.html)
- [Log record contents](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-logging.html)
- [CloudWatch metrics](https://docs.aws.amazon.com/network-firewall/latest/developerguide/monitoring-cloudwatch.html)
- [Query logs with Athena](https://docs.aws.amazon.com/athena/latest/ug/networkfirewall-example-queries.html)

---
