# AWS Shield Advanced - Detection, Health Checks & Visibility Cheat Sheet

## Overview

Shield Advanced improves DDoS **detection accuracy** by baselining your traffic and (optionally) incorporating **application health** via Route 53 health checks. It surfaces near-real-time attack diagnostics in the Shield console and via the API/CloudWatch, and offers a **global threat environment dashboard**.

**Key point:** Associating **Route 53 health checks** with your protections is the single biggest lever for reducing false positives — Shield uses app health to decide whether to mitigate more aggressively.

---

## Detection Inputs

| Input                        | Role                                                          |
| ---------------------------- | ------------------------------------------------------------- |
| **Traffic baselines**        | Shield learns normal patterns per protected resource          |
| **Route 53 health checks**   | Signals whether the app is actually degraded (health-based detection) |
| **Network/edge telemetry**   | AWS edge and infrastructure signals                           |
| **Protection groups**        | Aggregated detection across related resources                 |

---

## Health-Based Detection

Associating a Route 53 health check with a protection lets Shield correlate traffic anomalies with real application impact:

- If health is **healthy** during a traffic spike, Shield can be more conservative (avoid false-positive mitigation).
- If health is **unhealthy** during an anomaly, Shield can mitigate more aggressively and (with proactive engagement) the SRT can act.

```bash
# Associate a health check with a protection
aws shield associate-health-check \
  --protection-id <id> \
  --health-check-arn arn:aws:route53:::healthcheck/abc

# Disassociate
aws shield disassociate-health-check \
  --protection-id <id> --health-check-arn arn:aws:route53:::healthcheck/abc
```

---

## Visibility & Diagnostics

| Tool                          | Shows                                                        |
| ----------------------------- | ------------------------------------------------------------ |
| **Shield console events**     | Detected attacks, vectors, top talkers, duration            |
| **`list-attacks` / `describe-attack`** | Programmatic attack details                        |
| **CloudWatch metrics**        | `DDoSDetected`, attack volume metrics per protected resource |
| **Global threat dashboard**   | Broad DDoS activity across AWS                               |
| **Attack diagnostics**        | Vectors (SYN flood, UDP reflection, HTTP flood, etc.)        |

```bash
# List recent attacks in a time window
aws shield list-attacks \
  --start-time FromInclusive=2026-08-01T00:00:00Z \
  --end-time ToExclusive=2026-08-27T00:00:00Z

aws shield describe-attack --attack-id <id>

# Subscription-level attack summary
aws shield describe-attack-statistics
```

### Key CloudWatch metric

| Metric         | Meaning                                          |
| -------------- | ------------------------------------------------ |
| `DDoSDetected` | 1 when a DDoS is detected on a resource (alarm on it) |
| `DDoSAttackBitsPerSecond` / `...PacketsPerSecond` / `...RequestsPerSecond` | Attack volume |

---

## Gotchas & Caveats

1. **No health check = weaker detection** — without associated Route 53 health checks, Shield can't tell a real attack from a legitimate traffic surge, causing false positives or under-reaction.
2. **Baselines need time to form** — newly protected resources have less accurate detection until Shield learns normal patterns.
3. **Health checks must reflect real app health** — a health check that always passes (or checks the wrong thing) undermines health-based detection.
4. **`DDoSDetected` is per resource** — alarm per protected resource (or use protection groups) to catch attacks quickly.
5. **Detection is near-real-time, not instant** — there's a short delay before an attack registers.
6. **Diagnostics show vectors, not full packet capture** — use for triage/reporting, not forensic payloads.
7. **Global dashboard is informational** — it shows broad AWS DDoS activity, not necessarily targeting you.
8. **Health checks add cost** (Route 53 health-check pricing) but are worth it for critical resources.
9. **CloudWatch alarms need the right dimensions** (resource ARN) to be meaningful.
10. **Protection groups change aggregation** — a poorly chosen aggregation (SUM/MEAN/MAX) can mask a per-resource attack.
11. **Attack history retention** is limited — export/record important events for post-incident review.
12. **Proactive engagement depends on detection + health + contacts** — all three must be set up (see the SRT sheet).

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Too many false-positive mitigations         | No/weak health check                            | Associate an accurate Route 53 health check               |
| Attack not detected quickly                 | Baseline still forming / no alarm               | Allow baseline time; alarm on `DDoSDetected`              |
| Can't correlate anomaly with impact         | Health check checks wrong endpoint              | Point health check at a meaningful app endpoint           |
| No visibility into an attack                | Not looking at Shield events/metrics            | Use `list-attacks`/`describe-attack`; Shield console      |
| Group hides a single-resource attack        | Aggregation too coarse                          | Adjust protection-group aggregation/pattern               |
| Alarm never fires                           | Wrong CloudWatch dimensions                     | Alarm on `DDoSDetected` with correct resource dimension   |

---

## Best Practices

1. **Associate Route 53 health checks** with every important protection, pointing at meaningful endpoints.
2. **Alarm on `DDoSDetected`** per resource (or group) for fast awareness.
3. **Give baselines time** on new resources; expect lower accuracy initially.
4. **Use protection groups** thoughtfully to aggregate related resources without masking signal.
5. **Record attack diagnostics** for post-incident review and cost-protection claims.
6. **Combine with WAF rate-based rules** for concrete L7 enforcement while Shield detects.
7. **Set up proactive engagement** (SRT sheet) once health-based detection is in place.

---

## Useful Links

- [Health-based detection](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-health-checks.html)
- [Shield Advanced detection & mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-overview.html)
- [Viewing DDoS events](https://docs.aws.amazon.com/waf/latest/developerguide/using-ddos-reports.html)
- [Shield CloudWatch metrics](https://docs.aws.amazon.com/waf/latest/developerguide/monitoring-cloudwatch.html)
- [Global threat dashboard](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-overview.html)

---
