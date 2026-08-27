# AWS WAF - Logging & Monitoring Cheat Sheet

## Overview

WAF gives you three visibility mechanisms: **full request logging**, **CloudWatch metrics**, and **sampled requests**. Together they let you tune rules, investigate blocks, and prove coverage.

**Key point:** You cannot safely tune WAF without logging. Enable it before moving rules from Count to Block.

---

## Logging Destinations

WAF can send logs to three destination types:

| Destination                     | Best for                                       |
| ------------------------------- | ---------------------------------------------- |
| **CloudWatch Logs**             | Quick queries (Logs Insights), alarms          |
| **Amazon S3**                   | Cheap long-term storage, Athena analysis       |
| **Kinesis Data Firehose**       | Streaming to third parties / custom pipelines  |

### Requirements & naming

| Destination      | Naming / requirement                                            |
| ---------------- | --------------------------------------------------------------- |
| CloudWatch Logs  | Log group name **must start with** `aws-waf-logs-`              |
| S3 bucket        | Bucket name **must start with** `aws-waf-logs-`                 |
| Firehose         | Delivery stream name **must start with** `aws-waf-logs-`        |

> The `aws-waf-logs-` prefix is mandatory. Wrong prefix = logging config fails.

---

## Enabling Logging — CLI

```bash
# Enable logging to a CloudWatch Logs group
aws wafv2 put-logging-configuration \
  --logging-configuration \
    ResourceArn="arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc",\
LogDestinationConfigs="arn:aws:logs:us-east-1:123456789012:log-group:aws-waf-logs-my-acl" \
  --region us-east-1

# Get current logging config
aws wafv2 get-logging-configuration \
  --resource-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc" \
  --region us-east-1

# Disable logging
aws wafv2 delete-logging-configuration \
  --resource-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc" \
  --region us-east-1
```

---

## Log Field Filtering & Redaction

Reduce noise and protect sensitive data in the logging configuration:

| Feature               | Purpose                                                          |
| --------------------- | ---------------------------------------------------------------- |
| **RedactedFields**    | Mask fields (e.g., `Authorization` header, a password field)     |
| **LoggingFilter**     | Log only certain requests (e.g., only Blocked, or by label)      |

### LoggingFilter example (log only blocked + counted)

```json
{
  "LoggingFilter": {
    "DefaultBehavior": "DROP",
    "Filters": [
      {
        "Behavior": "KEEP",
        "Requirement": "MEETS_ANY",
        "Conditions": [
          { "ActionCondition": { "Action": "BLOCK" } },
          { "ActionCondition": { "Action": "COUNT" } }
        ]
      }
    ]
  }
}
```

> Filtering to only Block/Count/CAPTCHA events sharply cuts log volume (and cost) versus logging every Allowed request.

---

## Key Log Fields

| Field                      | Description                                            |
| -------------------------- | ------------------------------------------------------ |
| `action`                   | ALLOW / BLOCK / COUNT / CAPTCHA / CHALLENGE            |
| `terminatingRuleId`        | Rule that decided the outcome                          |
| `terminatingRuleType`      | REGULAR / RATE_BASED / MANAGED_RULE_GROUP             |
| `httpRequest.clientIp`     | Source IP                                              |
| `httpRequest.country`      | Geo of client                                          |
| `httpRequest.uri`          | Request path                                           |
| `httpRequest.args`         | Query string                                           |
| `httpRequest.headers`      | Request headers (unless redacted)                     |
| `ruleGroupList`            | Managed/rule-group evaluations & matches              |
| `rateBasedRuleList`        | Rate-based rules that matched                          |
| `labels`                   | Labels applied to the request                          |
| `captchaResponse` / `challengeResponse` | CAPTCHA/Challenge outcome                 |
| `timestamp`                | Event time (epoch ms)                                  |

---

## CloudWatch Metrics

WAF publishes metrics in the `AWS/WAFV2` namespace (per rule + per Web ACL, using your metric names).

| Metric                     | Meaning                                        |
| -------------------------- | ---------------------------------------------- |
| `AllowedRequests`          | Requests allowed                               |
| `BlockedRequests`          | Requests blocked                               |
| `CountedRequests`          | Requests matched a Count rule                  |
| `CaptchaRequests`          | Requests served a CAPTCHA                      |
| `ChallengeRequests`        | Requests served a Challenge                    |
| `RequestsWithValidCaptchaToken` | Passed CAPTCHA                            |
| `PassedRequests` / `SampleRequests` | (per rule visibility)                  |

> Enable `CloudWatchMetricsEnabled=true` in each rule's and the Web ACL's `VisibilityConfig` to get per-rule metrics.

### Example alarm (spike in blocks)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name waf-block-spike \
  --namespace AWS/WAFV2 \
  --metric-name BlockedRequests \
  --dimensions Name=WebACL,Value=my-web-acl Name=Region,Value=us-east-1 Name=Rule,Value=ALL \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 10000 --comparison-operator GreaterThanThreshold \
  --region us-east-1
```

---

## Sampled Requests

WAF stores a **sample** of requests each rule inspected (last few hours), viewable in console or CLI. Great for quick tuning without full logging.

```bash
aws wafv2 get-sampled-requests \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc" \
  --rule-metric-name AWSCommonRuleSet \
  --scope REGIONAL \
  --time-window StartTime=2026-08-27T00:00:00Z,EndTime=2026-08-27T01:00:00Z \
  --max-items 100 \
  --region us-east-1
```

| Aspect         | Detail                                             |
| -------------- | -------------------------------------------------- |
| Retention      | Recent window (few hours)                          |
| Sample size    | Up to a few hundred requests per query             |
| Enable via     | `SampledRequestsEnabled=true` in VisibilityConfig  |

---

## Querying Logs

### CloudWatch Logs Insights (top blocked IPs)

```
fields @timestamp, httpRequest.clientIp, action, terminatingRuleId
| filter action = "BLOCK"
| stats count(*) as blocks by httpRequest.clientIp
| sort blocks desc
| limit 20
```

### Blocked requests by rule

```
fields terminatingRuleId, action
| filter action = "BLOCK"
| stats count(*) as hits by terminatingRuleId
| sort hits desc
```

### Athena (S3 logs)

Create an external table over the S3 log path, then query by `action`, `terminatingruleid`, `httprequest.clientip`, etc. Partition by date for cost/performance.

---

## Firewall Manager & Multi-Account Visibility

| Tool                        | Use                                                       |
| --------------------------- | --------------------------------------------------------- |
| **Firewall Manager**        | Deploy WAF policies + centralized logging org-wide        |
| **Security Hub**            | Aggregate WAF-related findings                            |
| **Cross-account S3 logging**| Ship WAF logs to a central log-archive account            |

---

## Gotchas & Caveats

1. **The `aws-waf-logs-` prefix is mandatory** — the CloudWatch log group, S3 bucket, or Firehose stream name must start with it, or `put-logging-configuration` is rejected.
2. **Logging every Allowed request is expensive** — high-traffic sites generate huge log volume. Use a `LoggingFilter` to keep only BLOCK/COUNT/CAPTCHA events.
3. **Count actions only show up if you log them** — a rule in Count with no logging/metrics gives you nothing to tune from.
4. **Per-rule metrics need `CloudWatchMetricsEnabled=true`** in each rule's *and* the Web ACL's `VisibilityConfig` — it's not on automatically.
5. **Sampled requests are a small, short-lived sample** (recent hours) — good for quick tuning, not for audit or complete forensics. Use full logs for that.
6. **Redact sensitive fields explicitly** — Authorization headers, cookies, and password fields are logged unless you set `RedactedFields`.
7. **Log delivery isn't instant** — there's a short delay before events appear; don't assume "no logs yet" means "not matching."
8. **CloudWatch dimensions matter for alarms** — you must alarm on the right `WebACL`/`Region`/`Rule` dimensions or the alarm watches nothing.
9. **S3/Firehose logging incurs separate service costs** — ingestion, storage, and Firehose delivery are billed outside WAF.
10. **Cross-account logging needs permissions on the destination** — bucket policy / resource policy must allow WAF log delivery, or logs silently don't arrive.
11. **`terminatingRuleId` shows what decided the request** — for managed groups, drill into `ruleGroupList` to find the specific internal rule.
12. **Changing the logging destination doesn't backfill** — historical events aren't re-delivered to the new target.

### Official Caveats (from AWS docs)

1. **Redacted fields only affect the WAF logs.** The logging config's redacted-fields setting has **no impact on request sampling or Security Lake data collection**. To exclude fields from sampling or Security Lake, configure **Web ACL data protection** instead.
2. **Three destination types.** WAF logs can go to CloudWatch Logs, Amazon S3, or Amazon Data Firehose — each destination type has its own configuration specifics.
3. **AWS also keeps its own service logs.** Beyond the logs you enable, AWS uses service logs of the traffic WAF processes to support and protect customers/services.
4. **Security Lake collection is configured in Security Lake, not WAF**, and AWS doesn't charge a WAF fee for that option (Security Lake pricing still applies).
5. **Sampling is a separate feature.** Request sampling gives a view of evaluated traffic independently of full logging (and, per caveat 1, redaction doesn't apply to it).

> Source: [Logging AWS WAF web ACL traffic](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                     | Cause                                          | Fix                                                       |
| ----------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Logging config rejected                   | Destination name missing `aws-waf-logs-` prefix | Rename log group / bucket / stream with the prefix        |
| No logs appearing                         | Logging not enabled, or IAM/resource policy    | Verify `put-logging-configuration`; check permissions     |
| Log volume/cost too high                  | Logging every Allowed request                  | Add LoggingFilter to keep only Block/Count                |
| Sensitive data in logs                    | Auth headers / passwords logged                | Configure RedactedFields                                  |
| No per-rule metrics                       | `CloudWatchMetricsEnabled` false               | Enable in each rule's VisibilityConfig                    |
| Can't tell why a request was blocked      | Full logging off                               | Enable logging; inspect `terminatingRuleId` / labels      |
| Sampled requests empty                    | Sampling disabled or no matches in window      | Enable `SampledRequestsEnabled`; widen time window        |

---

## Best Practices

1. **Enable logging before enforcing** — tune from real data, not guesses.
2. **Use the `aws-waf-logs-` prefix** on every destination.
3. **Filter logs to Block/Count** to control volume and cost.
4. **Redact sensitive fields** (Authorization, password) for compliance.
5. **Alarm on block spikes and rate-rule triggers** to catch attacks/false positives.
6. **Centralize logs** in a dedicated account via Firewall Manager or cross-account S3.
7. **Use sampled requests for quick tuning**, full logs for investigations/audit.
8. **Partition Athena tables by date** for efficient long-term analysis.

---

## Useful Links

- [Logging Web ACL Traffic](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html)
- [Log Fields](https://docs.aws.amazon.com/waf/latest/developerguide/logging-fields.html)
- [Logging Destinations](https://docs.aws.amazon.com/waf/latest/developerguide/logging-destinations.html)
- [Log Filtering](https://docs.aws.amazon.com/waf/latest/developerguide/logging-management.html)
- [Redacting Fields](https://docs.aws.amazon.com/waf/latest/developerguide/logging-management.html#logging-management-redacting)
- [CloudWatch Metrics for WAF](https://docs.aws.amazon.com/waf/latest/developerguide/monitoring-cloudwatch.html)
- [Sampled Requests](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-testing.html)
- [Querying WAF Logs with Athena](https://docs.aws.amazon.com/athena/latest/ug/waf-logs.html)

---
