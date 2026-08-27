# Amazon VPC - Flow Logs Cheat Sheet

## Overview

**VPC Flow Logs** capture metadata about IP traffic going to and from network interfaces in your VPC. They record **connection metadata** (5-tuple, bytes, packets, action) — not packet payloads. Use them for security analysis, troubleshooting connectivity, and cost/traffic insight.

**Key point:** Flow logs record whether traffic was **ACCEPT**ed or **REJECT**ed by SG/NACL decisions — but only for the interface's own decisions, and they are near-real-time, not live.

---

## Where Flow Logs Attach

| Scope         | Captures                                                      |
| ------------- | ------------------------------------------------------------- |
| **VPC**       | All ENIs in the VPC (all subnets)                             |
| **Subnet**    | All ENIs in that subnet                                       |
| **ENI**       | A single network interface                                    |

> More-specific scopes don't override broader ones — you can have overlapping flow logs; each delivers independently.

---

## Destinations

| Destination                | Best for                                     |
| -------------------------- | -------------------------------------------- |
| **CloudWatch Logs**        | Quick queries (Logs Insights), alarms        |
| **Amazon S3**              | Cheap long-term storage, Athena analysis     |
| **Amazon Data Firehose**   | Streaming to third parties / pipelines       |

---

## Default Log Format (v2 fields, order)

```
version account-id interface-id srcaddr dstaddr srcport dstport
protocol packets bytes start end action log-status
```

| Field         | Meaning                                             |
| ------------- | --------------------------------------------------- |
| `srcaddr/dstaddr` | Source/destination IP                           |
| `srcport/dstport` | Ports                                           |
| `protocol`    | IANA protocol number (6=TCP, 17=UDP, 1=ICMP)        |
| `packets/bytes` | Volume in the aggregation window                  |
| `action`      | **ACCEPT** or **REJECT**                            |
| `log-status`  | OK / NODATA / SKIPDATA                               |

### Useful custom fields (v3–v5)

| Field                | Insight                                              |
| -------------------- | ---------------------------------------------------- |
| `vpc-id` / `subnet-id` / `instance-id` | Resource attribution              |
| `tcp-flags`          | SYN/ACK/FIN/RST — spot half-open scans               |
| `pkt-srcaddr` / `pkt-dstaddr` | Real endpoints behind NAT/intermediaries    |
| `flow-direction`     | ingress / egress                                     |
| `traffic-path`       | Egress path (IGW, NAT, peering, etc.)                |
| `action`             | ACCEPT / REJECT                                      |

---

## How It Works

```
Traffic on an ENI
        |
        v
Flow Logs aggregate over a window (default 10 min, or 1 min)
        |
        v
Records written to CloudWatch Logs / S3 / Firehose (near real-time, minutes of delay)
```

- **Aggregation interval:** default **10 minutes**, or set to **1 minute** for finer granularity.
- Records are **metadata**, sampled/aggregated — not every packet, no payload.

---

## CLI

```bash
# Flow log to CloudWatch Logs (needs an IAM role that can write logs)
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids vpc-0abc \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole \
  --max-aggregation-interval 60

# Flow log to S3 with custom fields + Parquet + hive-compatible partitions
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids vpc-0abc \
  --traffic-type REJECT \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::aws-flowlogs-bucket/prefix/ \
  --log-format '${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${action} ${tcp-flags} ${flow-direction}' \
  --destination-options FileFormat=parquet,HiveCompatiblePartitions=true,PerHourPartition=true
```

---

## CloudWatch Logs Insights examples

```
# Top rejected source IPs
fields srcaddr, dstaddr, dstport, action
| filter action = "REJECT"
| stats count(*) as hits by srcaddr
| sort hits desc | limit 20

# Top talkers by bytes
stats sum(bytes) as total by srcaddr, dstaddr
| sort total desc | limit 20
```

---

## Pricing

| Item                         | Cost                                              |
| ---------------------------- | ------------------------------------------------- |
| Flow Logs feature            | No separate charge for the feature                |
| Delivery/ingestion + storage | Charged by the destination (CloudWatch/S3/Firehose) — "vended logs" rates |

> Cost is driven by **log volume**. High-traffic VPCs generate large volumes; filter to REJECT or specific fields, and prefer S3 + Parquet for cheap analysis.

---

## Gotchas & Caveats

1. **Metadata only — no payloads.** Flow logs tell you *that* a connection happened, not *what* was sent. For packet capture use Traffic Mirroring.
2. **Not real-time.** Expect minutes of delay; the default aggregation is **10 minutes** (set 1 minute for faster).
3. **Some traffic is NOT logged** — e.g., traffic to the Amazon DNS server, DHCP, instance metadata (169.254.169.254), Windows license activation, and reserved-IP traffic.
4. **REJECT only reflects the layer that rejected** — a REJECT may be from a NACL or SG; flow logs alone don't always tell you which. Combine with config review.
5. **Stateful SG return traffic can look one-sided** — because SGs are stateful, you may see an ACCEPT without an obvious matching reverse record.
6. **`NODATA`/`SKIPDATA` statuses happen** — NODATA = no traffic in the window; SKIPDATA = some records dropped due to capacity/internal limits.
7. **Changing custom fields doesn't rewrite history** — a new format applies going forward only; Athena schemas must match the format used.
8. **You can't edit a flow log's format after creation** — delete and recreate to change fields.
9. **`pkt-srcaddr` vs `srcaddr`** differ behind NAT/intermediary — use the `pkt-` fields to see the true origin/destination.
10. **IAM/permissions on the destination are required** — CloudWatch needs a delivery role; S3/Firehose need bucket/stream policies, or delivery silently fails.
11. **Overlapping flow logs multiply cost** — VPC-level + subnet-level + ENI-level logs all bill separately.
12. **Cross-account delivery** to a central S3 bucket needs the right bucket policy allowing the log-delivery service.

---

## Troubleshooting

| Issue                                    | Cause                                          | Fix                                                        |
| ---------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| No logs appearing                        | Missing IAM role / bucket policy               | Grant delivery permissions; verify destination             |
| Expected traffic missing                 | It's in the not-logged list (DNS, metadata...) | Known limitation; use other tooling                        |
| Can't tell if SG or NACL rejected        | Flow logs don't name the layer                 | Review SG + NACL config; add fields; test                  |
| `SKIPDATA` in log-status                 | Internal capacity/aggregation limit            | Reduce scope or accept sampling; use 1-min interval        |
| Athena query returns garbage             | Table schema ≠ actual custom format            | Recreate table matching the exact field order              |
| Costs too high                           | Logging ALL traffic on a busy VPC              | Filter to REJECT; use S3+Parquet; avoid overlapping logs   |
| True client IP hidden behind NAT         | Using `srcaddr` not `pkt-srcaddr`              | Add/inspect `pkt-srcaddr`/`pkt-dstaddr`                    |

---

## Best Practices

1. **Enable at VPC level** for broad coverage; add ENI-level only when debugging specifics.
2. **Use S3 + Parquet + hive partitions** for cost-effective, queryable long-term storage.
3. **Add custom fields** (`tcp-flags`, `pkt-srcaddr`, `flow-direction`, `traffic-path`) for real investigative value.
4. **Set 1-minute aggregation** when troubleshooting; 10-minute for steady-state.
5. **Filter to REJECT** for security monitoring to cut volume, or ALL when you need full visibility.
6. **Centralize logs** cross-account into a security/log-archive account.
7. **Build standard Athena/Insights queries** (top talkers, top rejects) as a runbook.
8. **Remember it's metadata** — pair with Traffic Mirroring or Reachability Analyzer when you need packet or path detail.

---

## Useful Links

- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [Flow log records](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-records-examples.html)
- [Flow log record fields](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)
- [Publish to CloudWatch Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-cwl.html)
- [Publish to S3](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-s3.html)
- [Flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-limitations)
- [Query flow logs with Athena](https://docs.aws.amazon.com/athena/latest/ug/vpc-flow-logs.html)

---
