# AWS Transit Gateway - Monitoring & Flow Logs Cheat Sheet

## Overview

Transit Gateway visibility comes from **CloudWatch metrics** (per-TGW and per-attachment), **TGW Flow Logs** (connection metadata for traffic through the TGW), **Network Manager** (topology/route analysis), and **Reachability Analyzer / Route Analyzer** for path validation.

**Key point:** TGW Flow Logs are the TGW-level equivalent of VPC Flow Logs — metadata only, per-attachment context, delivered to CloudWatch/S3/Firehose.

---

## Monitoring Options

| Tool                      | Use                                                          |
| ------------------------- | ------------------------------------------------------------ |
| **CloudWatch metrics**    | Bytes/packets in/out, dropped packets, per-attachment usage  |
| **TGW Flow Logs**         | Connection metadata for traffic traversing the TGW           |
| **Network Manager**       | Global topology, events, route analysis across TGWs          |
| **Route Analyzer**        | Validate expected routing between attachments                |
| **Reachability Analyzer** | End-to-end path analysis (config-based)                      |

---

## Key CloudWatch Metrics (namespace `AWS/TransitGateway`)

| Metric                        | Meaning                                            |
| ----------------------------- | -------------------------------------------------- |
| `BytesIn` / `BytesOut`        | Traffic volume                                     |
| `PacketsIn` / `PacketsOut`    | Packet counts                                      |
| `PacketDropCountBlackhole`    | Packets dropped by blackhole routes                |
| `PacketDropCountNoRoute`      | Packets dropped for no matching route              |
| `BytesDropCount*`             | Byte-level drops                                    |

> `PacketDropCountNoRoute` and `PacketDropCountBlackhole` are the fastest signals for routing misconfig — alarm on them.

---

## TGW Flow Logs

Similar to VPC Flow Logs but with TGW-specific fields (attachment IDs, both source and destination attachment context).

| Field (representative)        | Meaning                                       |
| ----------------------------- | --------------------------------------------- |
| `srcaddr` / `dstaddr`         | Source/destination IP                         |
| `srcport` / `dstport`         | Ports                                          |
| `protocol`                    | IANA protocol number                          |
| `packets` / `bytes`           | Volume in the window                          |
| `tgw-id`                       | The Transit Gateway                           |
| `tgw-attachment-id`           | Attachment context                            |
| `resource-type`               | Attachment type                               |
| `packets-lost-no-route` / `packets-lost-blackhole` | Drop reasons          |

```bash
# Create TGW flow logs to S3
aws ec2 create-flow-logs \
  --resource-type TransitGateway \
  --resource-ids tgw-0abc \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::aws-tgw-flowlogs/prefix/ \
  --max-aggregation-interval 60

# Per-attachment flow logs
aws ec2 create-flow-logs \
  --resource-type TransitGatewayAttachment \
  --resource-ids tgw-attach-0abc \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /tgw/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole
```

---

## Pricing

| Item                    | Cost                                              |
| ----------------------- | ------------------------------------------------- |
| CloudWatch metrics      | Standard CloudWatch charges                        |
| TGW Flow Logs           | Vended-logs delivery + destination storage        |
| Network Manager         | Global Networks features may have charges          |

---

## Gotchas & Caveats

1. **TGW Flow Logs are metadata only** — no payloads; use them to see connection/drop patterns, not packet content.
2. **Aggregation delay** — like VPC Flow Logs, default 10-minute windows (set 1 minute for finer detail); not real-time.
3. **`traffic-type` for TGW is ALL** — TGW flow logs don't have the same ACCEPT/REJECT semantics as SG/NACL; use the drop-reason fields instead.
4. **Drops show up as `no-route` or `blackhole`** — distinguish intentional blackholes from missing routes when triaging.
5. **Per-attachment vs per-TGW logs** — attachment-level logs give finer attribution but multiply volume/cost.
6. **IAM/destination permissions required** — CloudWatch delivery role or S3/Firehose policy, or logs silently don't arrive.
7. **Custom fields require a matching Athena schema** — field order must match the configured format.
8. **Network Manager is global** — it aggregates across Regions/TGWs but needs a Global Network set up.
9. **Route Analyzer validates config, not live packets** — it tells you what routes *should* do, complementing flow logs.
10. **High-throughput TGWs generate large log volume** — filter/aggregate and prefer S3 + Parquet.
11. **Changing flow-log format doesn't backfill** — new fields apply going forward only.
12. **Metrics granularity** — per-attachment metrics help pinpoint which attachment drives traffic/drops; the TGW-level metric alone can mask the culprit.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Traffic dropped, cause unknown              | Blackhole vs no-route                           | Check `PacketDropCountBlackhole` vs `NoRoute` / flow-log fields |
| No flow logs delivered                      | Missing IAM role / bucket policy                | Grant delivery permissions                                 |
| Can't attribute traffic to an attachment    | Only TGW-level metrics enabled                  | Enable per-attachment metrics / flow logs                  |
| Route looks right but traffic fails         | Config vs runtime mismatch                      | Use Route Analyzer + flow logs together                    |
| Athena parse errors                         | Schema ≠ custom flow-log format                 | Recreate table matching field order                        |
| High logging cost                           | ALL traffic on busy TGW                         | Aggregate 10-min; S3 + Parquet; scope attachments          |

---

## Best Practices

1. **Alarm on `PacketDropCountNoRoute` and `PacketDropCountBlackhole`** to catch routing issues fast.
2. **Enable TGW Flow Logs to S3 (Parquet)** for cost-effective, queryable history.
3. **Add per-attachment logs/metrics** when you need to attribute or debug specific attachments.
4. **Use Route Analyzer** to validate expected reachability after routing changes.
5. **Set up Network Manager** for multi-Region topology and events.
6. **Use 1-minute aggregation** while troubleshooting, 10-minute steady-state.
7. **Centralize logs** cross-account for the networking/security team.

---

## Useful Links

- [TGW CloudWatch metrics](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-cloudwatch-metrics.html)
- [TGW Flow Logs](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-flow-logs.html)
- [TGW Flow Log records](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-flow-logs-records.html)
- [AWS Network Manager](https://docs.aws.amazon.com/network-manager/latest/tgwnm/what-are-global-networks.html)
- [Route Analyzer](https://docs.aws.amazon.com/network-manager/latest/tgwnm/route-analyzer.html)
- [Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/what-is-reachability-analyzer.html)

---
