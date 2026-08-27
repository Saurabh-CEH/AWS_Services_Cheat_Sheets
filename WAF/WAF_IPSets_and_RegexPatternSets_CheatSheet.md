# AWS WAF - IP Sets & Regex Pattern Sets Cheat Sheet

## Overview

**IP sets** and **regex pattern sets** are reusable resources you reference from rule statements. Define them once, reference them from many rules/Web ACLs, and update the set without editing each rule.

**Key point:** Referencing a set (vs inlining values) keeps rules simple and lets you update lists centrally — updating a set automatically updates every rule that references it.

---

## IP Sets

A named collection of IP addresses / CIDR ranges (IPv4 or IPv6). Used in `IPSetReferenceStatement`.

| Attribute        | Description                                          |
| ---------------- | ---------------------------------------------------- |
| **Name / ID / ARN** | Identifiers                                        |
| **IPAddressVersion** | `IPV4` or `IPV6` (one version per set)            |
| **Addresses**    | List of CIDRs (e.g., `203.0.113.0/24`, `2001:db8::/32`) |
| **Scope**        | REGIONAL or CLOUDFRONT                                |

### CIDR notes

| Input            | Meaning                          |
| ---------------- | -------------------------------- |
| `203.0.113.10/32`| Single IPv4 address              |
| `203.0.113.0/24` | 256-address IPv4 range           |
| `0.0.0.0/0`      | **Not supported** — WAF rejects `/0` in an IP set |
| `2001:db8::/32`  | IPv6 range                       |

> A single IP set holds **one** address version. Need both? Create an IPv4 set and an IPv6 set and reference both (OR statement).

---

### IP Set — CLI

```bash
# Create an IPv4 IP set
aws wafv2 create-ip-set \
  --name blocked-ips \
  --scope REGIONAL \
  --region us-east-1 \
  --ip-address-version IPV4 \
  --addresses "203.0.113.0/24" "198.51.100.10/32"

# Get the IP set (returns LockToken needed for updates)
aws wafv2 get-ip-set \
  --name blocked-ips --scope REGIONAL \
  --id 11111111-2222-3333-4444-555555555555 \
  --region us-east-1

# Update the IP set (REPLACES the full list — include ALL addresses to keep)
aws wafv2 update-ip-set \
  --name blocked-ips --scope REGIONAL \
  --id 11111111-2222-3333-4444-555555555555 \
  --lock-token abcd-1234-... \
  --addresses "203.0.113.0/24" "198.51.100.10/32" "192.0.2.55/32" \
  --region us-east-1

# List IP sets
aws wafv2 list-ip-sets --scope REGIONAL --region us-east-1

# Delete
aws wafv2 delete-ip-set \
  --name blocked-ips --scope REGIONAL \
  --id 11111111-2222-3333-4444-555555555555 \
  --lock-token abcd-1234-... \
  --region us-east-1
```

> **`update-ip-set` replaces the entire address list.** Always send the complete desired set, not just additions, or you'll drop existing entries.

---

### IP Set — Reference in a Rule

```json
{
  "Name": "BlockBadIPs",
  "Priority": 0,
  "Statement": {
    "IPSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/ipset/blocked-ips/1111..."
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockBadIPs"
  }
}
```

### Using the forwarded IP (behind CDN/proxy)

When WAF sits behind a proxy, the real client IP is in a header. Configure `IPSetForwardedIPConfig`:

```json
{
  "IPSetReferenceStatement": {
    "ARN": "arn:aws:wafv2:...:ipset/allowlist/...",
    "IPSetForwardedIPConfig": {
      "HeaderName": "X-Forwarded-For",
      "FallbackBehavior": "MATCH",
      "Position": "FIRST"
    }
  }
}
```

| Field             | Values                          |
| ----------------- | ------------------------------- |
| **HeaderName**    | e.g., `X-Forwarded-For`         |
| **Position**      | FIRST / LAST / ANY              |
| **FallbackBehavior** | MATCH / NO_MATCH (if header absent) |

---

## Regex Pattern Sets

A named collection of regular expressions. Used in `RegexPatternSetReferenceStatement`.

| Attribute            | Description                                  |
| -------------------- | -------------------------------------------- |
| **Name / ID / ARN**  | Identifiers                                  |
| **RegularExpressionList** | List of regex strings                   |
| **Scope**            | REGIONAL or CLOUDFRONT                        |

> WAF uses a **PCRE-like** regex engine with restrictions (no arbitrary backreferences/lookbehind in some cases). Keep patterns simple to avoid rejection and control WCU.

---

### Regex Pattern Set — CLI

```bash
# Create a regex pattern set (bad user agents)
aws wafv2 create-regex-pattern-set \
  --name bad-user-agents \
  --scope REGIONAL \
  --region us-east-1 \
  --regular-expression-strings '[{"RegexString":"(?i)sqlmap"},{"RegexString":"(?i)nikto"},{"RegexString":"(?i)masscan"}]'

# Get (returns LockToken)
aws wafv2 get-regex-pattern-set \
  --name bad-user-agents --scope REGIONAL \
  --id aaaa-bbbb-cccc-dddd \
  --region us-east-1

# Update (replaces full list)
aws wafv2 update-regex-pattern-set \
  --name bad-user-agents --scope REGIONAL \
  --id aaaa-bbbb-cccc-dddd \
  --lock-token abcd-1234-... \
  --regular-expression-strings '[{"RegexString":"(?i)sqlmap"},{"RegexString":"(?i)nikto"}]' \
  --region us-east-1

# List / Delete
aws wafv2 list-regex-pattern-sets --scope REGIONAL --region us-east-1
aws wafv2 delete-regex-pattern-set \
  --name bad-user-agents --scope REGIONAL \
  --id aaaa-bbbb-cccc-dddd \
  --lock-token abcd-1234-... \
  --region us-east-1
```

---

### Regex Pattern Set — Reference in a Rule

```json
{
  "Name": "BlockScanners",
  "Priority": 1,
  "Statement": {
    "RegexPatternSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/regexpatternset/bad-user-agents/aaaa...",
      "FieldToMatch": { "SingleHeader": { "Name": "user-agent" } },
      "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockScanners"
  }
}
```

> `RegexMatchStatement` (inline single regex) also exists for one-off patterns. Use a **pattern set** when the list is shared or grows.

---

## IP Set vs Regex Set — When to Use

| Need                                     | Use                    |
| ---------------------------------------- | ---------------------- |
| Allow/block specific IPs or ranges       | IP set                 |
| Match scanner tools / UA patterns        | Regex pattern set      |
| Match URI/query patterns dynamically     | Regex pattern set      |
| Central allowlist for partners           | IP set (referenced everywhere) |
| One-off single pattern                   | Inline RegexMatch/ByteMatch |

---

## Quotas

| Resource                                | Default limit         |
| --------------------------------------- | --------------------- |
| IP sets per account per region          | 100                   |
| IP addresses/CIDRs per IP set           | 10,000                |
| Regex pattern sets per account per region | 10                  |
| Regex expressions per pattern set       | 10                    |
| Regex string length                     | 512 chars             |

> Need more entries per set? Some limits are adjustable via Service Quotas. For very large IP lists, consider splitting across sets and OR-combining, or using Firewall Manager.

---

## Gotchas & Caveats

1. **`update-ip-set` and `update-regex-pattern-set` REPLACE the entire list** — they do not append. Always send the full desired set or you'll silently drop entries.
2. **An IP set holds one address version** — IPv4 or IPv6, never both. Dual-stack needs two sets OR'd together.
3. **A `/32` (or `/128`) is required for a single host** — a bare IP without a mask is not accepted; always use CIDR notation.
4. **Behind a proxy/CDN the set matches the wrong IP** — without `IPSetForwardedIPConfig`, you match the edge IP, so allow/block lists target the CDN, not the client.
5. **`FallbackBehavior` decides missing-header cases** — if `X-Forwarded-For` is absent, MATCH vs NO_MATCH changes the outcome; pick intentionally.
6. **Forwarded-IP can be spoofed** — only trust the header behind a controlled proxy/CDN that overwrites it.
7. **Regex engine is restricted** — unsupported PCRE features are rejected at create/update time; keep patterns simple.
8. **Case sensitivity bites regex** — add a LOWERCASE transformation or `(?i)`; otherwise `Sqlmap` slips past `sqlmap`.
9. **Only 10 regex expressions per pattern set, 512 chars each** — large signature libraries need multiple sets OR'd, or a different approach.
10. **You cannot delete a set that's still referenced** — remove it from every rule/Web ACL first (`WAFAssociatedItemException`).
11. **Lock tokens apply to sets too** — updates need the current `LockToken`; automation must handle `WAFOptimisticLockException`.
12. **Heavy/backtracking regex inflates WCU** and can slow evaluation — avoid catastrophic patterns.

### Official Caveats (from AWS docs)

**IP sets**
1. **All IPv4 and IPv6 CIDRs are supported except `/0`.** You cannot use a `/0` range in an IP set.
2. **An IP set holds up to 10,000 addresses or ranges.**
3. **Sets are maintained independently of rules.** One set can be used in many rules, and updating the set **automatically updates every rule that references it**.
4. **Base cost is 1 WCU**, but a forwarded-IP configuration with position **`ANY` adds 4 WCU**.
5. **Forwarded IP config needs a fallback behavior** for malformed IPs in the header (match or no match).

**Regex pattern sets**
1. **PCRE-based syntax (libpcre) with exceptions.** WAF supports the PCRE pattern syntax with some exceptions — check the supported-syntax reference before relying on advanced features.
2. **Base cost is 25 WCU** per regex pattern set match statement. Add **10 WCU** for `All query parameters`, **double** the base for `JSON body`, and **10 WCU per text transformation**.
3. **Matches if ANY pattern in the set matches.** To combine patterns with logic (match some, not others), use a `RegexMatch` statement instead.
4. **Sets are maintained independently of rules** and updating a set **auto-updates all referencing rules**.

> Sources: [IP set match rule statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-ipset-match.html) and [Regex pattern set match rule statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-regex-pattern-set-match.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                    | Cause                                        | Fix                                                        |
| ---------------------------------------- | -------------------------------------------- | ---------------------------------------------------------- |
| IP set update dropped existing entries   | `update-ip-set` replaces the full list       | Always send the complete desired address list             |
| Rule using set matches wrong client IP   | Behind proxy/CDN; matching edge IP            | Use `IPSetForwardedIPConfig` with X-Forwarded-For          |
| `WAFOptimisticLockException`             | Stale lock token                              | Re-`get` the set, use fresh LockToken                      |
| Regex rejected on create/update          | Unsupported regex feature                     | Simplify pattern; avoid unsupported lookbehind/backrefs    |
| Regex not matching case variants         | No LOWERCASE transform / no `(?i)`            | Add LOWERCASE transformation or `(?i)` flag                |
| Cannot delete set                        | Still referenced by a rule                    | Remove references from all Web ACLs first                  |

---

## Best Practices

1. **Reference sets, don't inline** — central updates, lower WCU, cleaner rules.
2. **Always send the full list on updates** — updates replace, not append.
3. **Use forwarded-IP config** behind CloudFront/ALB/proxies.
4. **Keep regex simple** — case-insensitive flag + LOWERCASE, avoid heavy backtracking.
5. **Separate IPv4 and IPv6 sets** and OR them in rules.
6. **Automate set updates** (e.g., Lambda pulling a threat feed) with lock-token handling.
7. **Cache lock tokens** and refresh on conflict in automation.
8. **Document what each set is for** — a blocklist and an allowlist should never be confused.

---

## Useful Links

- [Creating an IP Set](https://docs.aws.amazon.com/waf/latest/developerguide/waf-ip-set-creating.html)
- [IP Set Match Statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-ipset-match.html)
- [Using Forwarded IP Addresses](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-forwarded-ip-address.html)
- [Creating a Regex Pattern Set](https://docs.aws.amazon.com/waf/latest/developerguide/waf-regex-pattern-set-creating.html)
- [Regex Match Statement](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-regex-pattern-set-match.html)
- [Supported Regex Syntax](https://docs.aws.amazon.com/waf/latest/developerguide/waf-regex-pattern-set-creating.html)
- [AWS WAF Quotas](https://docs.aws.amazon.com/waf/latest/developerguide/limits.html)

---
