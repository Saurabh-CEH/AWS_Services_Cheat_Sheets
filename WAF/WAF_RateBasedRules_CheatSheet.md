# AWS WAF - Rate-Based Rules Cheat Sheet

## Overview

A **rate-based rule** tracks the request rate from an aggregation key (by default the client IP) over a sliding time window. When a key exceeds the configured limit, the rule's action (usually Block or Challenge) applies until the rate drops back below the limit.

**Key point:** Rate-based rules are the primary WAF tool against **Layer 7 (application-layer) floods**, brute-force, and scraping. They complement — not replace — Shield's L3/L4 protection.

---

## How It Works

```
Requests arrive continuously
        |
        v
WAF counts requests per aggregation key over the EVALUATION WINDOW
        |
        ├── count <= limit  → rule does NOT match (traffic passes)
        |
        └── count >  limit  → rule MATCHES → apply action (Block/Challenge/CAPTCHA/Count)
                                 |
                                 v
              Action stays applied for that key until its rate falls below the limit
```

- The count is a **rolling estimate** over the window, not a hard per-second cap.
- WAF estimates the rate; there is a short delay before blocking starts and before it stops.

---

## Key Settings

| Setting                    | Description                                                        |
| -------------------------- | ------------------------------------------------------------------ |
| **Limit**                  | Max requests allowed per key within the window                     |
| **EvaluationWindowSec**    | 60, 120, 300 (default), or 600 seconds                             |
| **AggregateKeyType**       | How to group requests (IP, FORWARDED_IP, CUSTOM_KEYS, CONSTANT)    |
| **ScopeDownStatement**     | Only count requests matching this condition                        |
| **Action**                 | Block / Count / CAPTCHA / Challenge                                |

### Limit range

- Minimum limit: **100** requests per window.
- Maximum limit: **2,000,000,000** requests per window.

---

## Aggregation Key Types

| AggregateKeyType | Aggregates by                                                         |
| ---------------- | --------------------------------------------------------------------- |
| **IP**           | Source IP seen by WAF (default)                                       |
| **FORWARDED_IP** | IP from a header like `X-Forwarded-For` (behind a proxy/CDN)          |
| **CONSTANT**     | All requests as one bucket (global rate limit for a path)             |
| **CUSTOM_KEYS**  | Combination of up to 5 keys (IP, header, cookie, query arg, label...) |

### Custom keys (WAFv2)

You can aggregate on combinations such as **IP + session cookie**, **IP + URI path**, or a **header value alone** — great for per-user or per-endpoint limits.

| Custom key source | Example use                                    |
| ----------------- | ---------------------------------------------- |
| IP                | Per client IP                                  |
| HTTPHeader        | Per `Authorization` / API key                  |
| Cookie            | Per session cookie                             |
| QueryArgument     | Per `userId` param                             |
| UriPath           | Per endpoint (combine with IP)                 |
| LabelNamespace    | Per label added by an earlier rule             |

---

## Common Patterns

### 1. Basic per-IP flood protection

```json
{
  "RateBasedStatement": {
    "Limit": 2000,
    "EvaluationWindowSec": 300,
    "AggregateKeyType": "IP"
  }
}
```

Blocks any IP exceeding 2,000 requests in 5 minutes.

### 2. Protect only the login endpoint (scope-down)

```json
{
  "RateBasedStatement": {
    "Limit": 100,
    "EvaluationWindowSec": 60,
    "AggregateKeyType": "IP",
    "ScopeDownStatement": {
      "ByteMatchStatement": {
        "SearchString": "/login",
        "FieldToMatch": { "UriPath": {} },
        "PositionalConstraint": "STARTS_WITH",
        "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
      }
    }
  }
}
```

### 3. Per-user limit via custom key (IP + session cookie)

```json
{
  "RateBasedStatement": {
    "Limit": 300,
    "EvaluationWindowSec": 300,
    "AggregateKeyType": "CUSTOM_KEYS",
    "CustomKeys": [
      { "IP": {} },
      { "Cookie": { "Name": "session", "TextTransformations": [ { "Priority": 0, "Type": "NONE" } ] } }
    ]
  }
}
```

### 4. Global limit for an expensive endpoint (CONSTANT)

```json
{
  "RateBasedStatement": {
    "Limit": 5000,
    "EvaluationWindowSec": 300,
    "AggregateKeyType": "CONSTANT",
    "ScopeDownStatement": {
      "ByteMatchStatement": {
        "SearchString": "/report/export",
        "FieldToMatch": { "UriPath": {} },
        "PositionalConstraint": "STARTS_WITH",
        "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
      }
    }
  }
}
```

---

## Action Choices

| Action        | When to use                                                       |
| ------------- | ----------------------------------------------------------------- |
| **Block**     | Straightforward flood mitigation                                  |
| **Count**     | Baseline/tuning before enforcing                                  |
| **Challenge** | Suspected bots — let real browsers pass silently                  |
| **CAPTCHA**   | Interactive endpoints where a human check is acceptable           |

> For scraping/bot floods, **Challenge** often beats Block: legit browsers transparently pass, headless bots fail.

---

## Choosing the Window & Limit

| Goal                        | Window | Limit guidance                              |
| --------------------------- | ------ | ------------------------------------------- |
| Brute-force login defense   | 60s    | Low (e.g., 100) on `/login` via scope-down  |
| General flood protection    | 300s   | Above your peak legit per-IP traffic        |
| Burst-tolerant APIs         | 600s   | Higher, smooths spikes                      |

**Sizing tip:** review CloudWatch/logs for legitimate per-IP peaks, then set the limit comfortably above that to avoid false positives.

---

## Interaction with Other Rules

- Rate-based rules obey **priority** like any rule; a higher-priority Allow can exempt trusted IPs.
- Combine with an **IP set allow rule at lower priority** to exclude partners/monitoring from rate limits.
- Custom key aggregation can reference **labels** from earlier managed rules (e.g., only rate-limit unverified bots).

```
Priority 0:  Allow  → IP set (monitoring + partners)
Priority 1:  Block  → Rate-based (IP, 2000 / 5 min)
Default: ALLOW
```

---

## CLI / Monitoring

```bash
# Rate-based rules live inside the Web ACL rules array (update-web-acl).

# See how many distinct keys are currently rate-limited
aws wafv2 get-rate-based-statement-managed-keys \
  --scope REGIONAL \
  --region us-east-1 \
  --web-acl-name my-web-acl \
  --web-acl-id 12345678-1234-1234-1234-123456789012 \
  --rule-name RateLimitRule
```

Key CloudWatch metrics: `BlockedRequests`, `CountedRequests`, `AllowedRequests` (filter by the rule's metric name).

---

## Quotas & Limits

| Item                                   | Limit                       |
| -------------------------------------- | --------------------------- |
| Rate-based rules per Web ACL           | 10 (default)                |
| Minimum limit value                    | 100 requests / window       |
| Maximum limit value                    | 2,000,000,000 / window      |
| Evaluation windows supported           | 60, 120, 300, 600 seconds   |
| Custom keys per rate-based rule        | up to 5                     |
| WCU per rate-based rule                | ~2 (plus scope-down cost)   |

---

## Gotchas & Caveats

1. **The rate is a rolling estimate, not a hard cap** — blocking starts *after* the limit is exceeded and stops *after* the rate drops, with a short lag. Don't expect precise per-second enforcement.
2. **Default aggregation is the IP WAF sees** — behind CloudFront/ALB/proxy that's the edge/proxy IP, so all users can share one bucket. Use FORWARDED_IP or custom keys.
3. **Minimum limit is 100 per window** — you can't rate-limit below that; for stricter control combine with other conditions.
4. **Without a scope-down, it counts all traffic** — including static assets and health checks, so a real limit for the app path may be wrong for the whole site.
5. **`CONSTANT` aggregation is a single global bucket** — powerful for one endpoint, but one noisy client can trip it for everyone. Use deliberately.
6. **Custom keys are capped at 5** — and high-cardinality keys (per-user) increase tracking overhead; keep combinations minimal.
7. **Only 10 rate-based rules per Web ACL by default** — consolidate with custom keys or request a quota increase.
8. **Blocked keys are evicted over time** — the managed-keys list is a snapshot; an IP dropping below the limit is released automatically.
9. **A higher-priority Allow bypasses the rate rule** — trusted-IP allowlists placed before the rate rule exempt those sources entirely (usually intended, but easy to forget).
10. **FORWARDED_IP trusts a header** — if clients can spoof `X-Forwarded-For`, they can evade or poison aggregation. Only trust it behind a controlled proxy/CDN.
11. **CAPTCHA/Challenge actions on rate rules are billed** and require client JS — API/native clients may fail them.
12. **Rate rules react in ~minutes, not instantly** — for fast volumetric floods, pair with Shield Advanced.

### Official Caveats (from AWS docs)

AWS documents these caveats explicitly for rate-based rules. WAF rate limiting is designed to protect availability efficiently — it is **not** intended for precise request-rate limiting.

1. **Not an exact limit.** WAF estimates the current request rate with an algorithm that weights more recent requests more heavily. It applies rate limiting *near* the limit you set, but does not guarantee an exact match to it.
2. **Detection lag at both ends (usually under 30s).** Because WAF looks back over the configured evaluation window (and due to propagation delays), requests can come in over the limit for **up to several minutes** before WAF detects and rate-limits them. Likewise, the rate can drop below the limit for a period before WAF stops the rate-limiting action. This delay is usually below 30 seconds.
3. **Changing settings resets counts and can pause limiting for up to a minute.** Editing any rate-limit setting on an in-use rule resets the rule's rate-limiting counts and can pause its rate-limiting for up to a minute. The settings that trigger this reset are: **evaluation window, rate limit, request aggregation settings, forwarded IP configuration, and scope of inspection.**

> Source: [Rate-based rule caveats in AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based-caveats.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                    | Cause                                             | Fix                                                        |
| ---------------------------------------- | ------------------------------------------------- | ---------------------------------------------------------- |
| Legit users blocked during traffic spike | Limit too low for real peak                       | Raise limit; use larger window; add allow rule for trusted |
| Blocking starts/stops with delay         | Rate is a rolling estimate                        | Expected behavior; tighten window for faster reaction      |
| All users share one IP (behind proxy)    | AggregateKeyType=IP sees the proxy IP             | Use FORWARDED_IP or CUSTOM_KEYS (cookie/header)            |
| Rate rule counts unrelated traffic       | No scope-down                                     | Add ScopeDownStatement to target the endpoint             |
| Rule never triggers                      | Limit far above actual traffic, or in Count       | Lower limit; switch action to Block                        |
| Too many rate rules needed               | Hitting 10-rule limit                             | Consolidate with custom keys; request quota increase       |

---

## Best Practices

1. **Baseline in Count first** to size the limit against real traffic.
2. **Scope down** to sensitive/expensive endpoints instead of the whole site.
3. **Protect login/auth separately** with a tight window and low limit.
4. **Use Challenge for bot floods** so real browsers aren't disrupted.
5. **Exempt trusted sources** with a higher-priority Allow + IP set.
6. **Behind a CDN/proxy? Use FORWARDED_IP or custom keys**, not raw IP.
7. **Combine with Shield Advanced** for volumetric + application-layer coverage.
8. **Set custom 429 responses** so clients can back off gracefully.

---

## Useful Links

- [Rate-Based Rule Statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)
- [Rate-Based Rule Aggregation Options](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rate-based-example-limit-login-page-keys.html)
- [Custom Aggregation Keys](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rate-based-rule-high-cardinality.html)
- [Rate-Based Rule Behavior](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based-behavior.html)
- [get-rate-based-statement-managed-keys](https://docs.aws.amazon.com/cli/latest/reference/wafv2/get-rate-based-statement-managed-keys.html)

---
