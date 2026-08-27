# AWS WAF - Pricing, Quotas & Troubleshooting Cheat Sheet

## Overview

This sheet consolidates **cost drivers**, **service limits**, and the **most common operational issues** across AWS WAF (WAFv2). Use it for capacity planning, quota requests, and fast incident triage.

**Key point:** WAF cost has three base components (Web ACL, rules, requests) plus **extra usage-based fees** for intelligent features (Bot Control, ATP, ACFP).

---

## Pricing Model

| Component                        | Typical charge                                   |
| -------------------------------- | ------------------------------------------------ |
| **Web ACL**                      | ~$5.00 per month (per Web ACL)                   |
| **Rule**                         | ~$1.00 per month (per rule in a Web ACL)         |
| **Managed rule group**           | Counts as rules; some intelligent ones add fees  |
| **Requests**                     | ~$0.60 per 1 million requests                    |
| **Bot Control**                  | Monthly fee + per-request inspected              |
| **Account Takeover Prevention**  | Monthly fee + per-login-request inspected        |
| **Account Creation Fraud Prev.** | Monthly fee + per-registration-request inspected |
| **CAPTCHA**                      | Per attempt/challenge served                     |
| **Bot Control / Fraud Control**  | Additional per-request analysis fees             |

> Prices vary by region and change over time — confirm on the [pricing page](https://aws.amazon.com/waf/pricing/). Base managed rule groups (CRS, KnownBadInputs, IP reputation) have **no extra rule-group fee** beyond normal rule pricing.

### What drives cost most

1. **Request volume** — the per-million-request charge scales with traffic.
2. **Intelligent features** — Bot Control/ATP/ACFP add per-request analysis fees.
3. **Number of rules** — each rule is a monthly line item.
4. **Logging** — CloudWatch/S3/Firehose ingestion & storage (separate service costs).

### Cost-control levers

- **Scope-down statements** so expensive groups only inspect relevant paths.
- **LoggingFilter** to log only Block/Count events.
- **Consolidate rules** and remove unused managed groups.
- **Use rate-based rules** (cheap) instead of intelligent groups for pure volumetric floods.

---

## Quotas (Defaults)

### Web ACL & rules

| Resource                                   | Default limit         | Adjustable |
| ------------------------------------------ | --------------------- | ---------- |
| Web ACLs per account per region            | 100                   | Yes        |
| Rules per Web ACL                          | (bounded by WCU)      | via WCU    |
| WCUs per Web ACL                           | 1,500                 | Yes        |
| Rate-based rules per Web ACL               | 10                    | Yes        |
| Rule groups per account per region         | 100                   | Yes        |
| Rules per rule group                       | (bounded by WCU)      | via WCU    |
| Referenced rule groups per Web ACL         | (bounded by WCU)      | via WCU    |

### Sets

| Resource                                   | Default limit         | Adjustable |
| ------------------------------------------ | --------------------- | ---------- |
| IP sets per account per region             | 100                   | Yes        |
| IP addresses/CIDRs per IP set              | 10,000                | Yes        |
| Regex pattern sets per account per region  | 10                    | Yes        |
| Regex expressions per pattern set          | 10                    | Yes        |
| Regex string length                        | 512 characters        | No         |

### Other

| Item                                       | Default limit         |
| ------------------------------------------ | --------------------- |
| Body inspected (CloudFront, API GW, Cognito, App Runner, Verified Access, Bedrock AgentCore) | 16 KB default, up to 64 KB |
| Body inspected (ALB, AppSync)              | 8 KB (fixed)          |
| Headers / Cookies inspected                | first 8 KB AND first 200 headers/cookies |
| Text transformations per field             | 10                    |
| Custom keys per rate-based rule            | 5                     |
| Logging destinations per Web ACL           | (per destination limits) |

> Most numeric limits are adjustable through **Service Quotas** / a support request. WCU cap increases are the most common request when stacking managed groups.

---

## WCU Budget Quick Reference

| Statement / group             | Approx WCU |
| ----------------------------- | ---------- |
| IP set / Geo match            | 1          |
| Byte match                    | 1 (+transforms) |
| Regex pattern set             | 25 (base)  |
| SQLi / XSS                    | ~10 – 20   |
| Rate-based rule               | 2          |
| CommonRuleSet (CRS)           | ~700       |
| KnownBadInputs / SQLi set     | ~200       |
| IP reputation list            | ~25        |
| AnonymousIpList               | ~50        |
| Bot Control / ATP / ACFP      | ~50 each   |

> CRS alone consumes nearly half the default 1,500 WCU budget. Plan before adding more.

---

## Common Errors (API)

| Error                              | Meaning / cause                              | Fix                                                    |
| ---------------------------------- | -------------------------------------------- | ------------------------------------------------------ |
| `WAFOptimisticLockException`       | Stale lock token on update/delete            | Re-`get` the resource, retry with fresh LockToken      |
| `WAFUnavailableEntityException`    | Resource in use (e.g., Web ACL still associated) | Disassociate/remove references first               |
| `WAFLimitsExceededException`       | Hit WCU or resource quota                     | Trim rules or request a quota increase                 |
| `WAFNonexistentItemException`      | Wrong ID/name/scope/region                    | Verify identifiers and scope/region                    |
| `WAFDuplicateItemException`        | Name already exists                           | Use a unique name                                      |
| `WAFInvalidParameterException`     | Bad statement/config JSON                     | Validate JSON structure and field values              |
| `WAFAssociatedItemException`       | Deleting a set/group still referenced         | Remove references from all Web ACLs first              |

---

## Common Operational Issues

| Symptom                                   | Likely cause                                   | Fix                                                       |
| ----------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Legit users get **403**                   | Managed rule false positive (often `SizeRestrictions_BODY`) | Override the specific rule to Count; check logs |
| File/Excel upload blocked                 | Body size restriction / content-type rules     | Override size rule; raise body inspection limit           |
| Blocked users from a country you allow    | Geo rule misconfig or edge-IP geo mismatch      | Verify GeoMatch codes; check forwarded IP                 |
| Rate rule blocks real traffic spike       | Limit too low                                   | Raise limit / widen window; allowlist trusted IPs         |
| Rule "not working"                        | Higher-priority Allow, or OverrideAction=Count  | Fix priorities; set OverrideAction=None                   |
| Country shows as `-`/unknown in logs      | WAF can't geolocate the source IP               | Check X-Forwarded-For handling behind CDN                 |
| Everything blocked                        | Default action = Block with no Allow rules      | Add Allow rules or set default action to Allow            |
| Cannot attach to NLB / Classic LB         | Unsupported resource                            | Front with ALB or CloudFront                              |
| Custom WebACL missing in member accounts (FMS) | Firewall Manager policy scope/config issue | Review FMS policy scope, remediation, and account tags    |
| No visibility into blocks                 | Logging disabled                                | Enable logging + sampled requests                         |

---

## Gotchas & Caveats

1. **You pay per request even for Allowed traffic** — the per-million-request charge applies to all inspected requests, not just blocks. High-traffic sites are dominated by this line item.
2. **Intelligent features (Bot Control/ATP/ACFP) add per-request analysis fees** on top of base WAF cost — without scope-down you pay to inspect everything.
3. **Each rule is a monthly charge** — leftover experimental/Count rules quietly cost money; clean them up.
4. **WCU is the real limiter, not "rule count"** — the 1,500 default cap is consumed by complexity; CommonRuleSet alone is ~700.
5. **Most limits are soft, but not all** — regex string length (512) and some structural limits are fixed; don't design around raising them.
6. **`WAFOptimisticLockException` is expected under concurrency** — automation must re-`get` and retry, not treat it as fatal.
7. **Deleting a set/group that's still referenced fails** — remove all references first (`WAFAssociatedItemException` / `WAFUnavailableEntityException`).
8. **Country shows as `-`/unknown for some IPs** — WAF geolocation isn't universal; behind a proxy it geolocates the proxy unless forwarded-IP is set.
9. **A Block-by-default Web ACL with no Allow rules blocks everything** — a classic self-inflicted outage.
10. **Logging is a separate cost** — CloudWatch/S3/Firehose ingestion and storage are billed by those services, not WAF.
11. **FMS-managed Web ACLs can't be edited directly in member accounts** — changes must go through the Firewall Manager policy, or they get reverted.
12. **Orphaned Classic (v1) resources keep billing** — check both `waf`/`waf-regional` and `wafv2` when auditing cost.

---

## Incident Triage Flow

```
Users report being blocked
        |
        v
1. Check WAF logs → find action=BLOCK, note terminatingRuleId
        |
        v
2. Is it a managed rule?  → override that specific rule to Count, retest
        |
        v
3. Is it a rate-based rule? → check limit vs traffic; allowlist if legit
        |
        v
4. Is it geo/IP?  → verify country codes / forwarded-IP config
        |
        v
5. Confirm fix in Count + sampled requests → re-enable Block
```

---

## Best Practices

1. **Track your WCU budget** — know how much headroom you have before adding groups.
2. **Request quota increases proactively** (WCU, rate rules, IP set size) before you hit them.
3. **Control cost with scope-down + logging filters**, especially for intelligent groups.
4. **Cache lock tokens** and handle `WAFOptimisticLockException` with retry.
5. **Keep a runbook** mapping common `terminatingRuleId`s to their fixes.
6. **Override noisy managed rules to Count**, don't disable whole groups.
7. **Always be able to answer "why was this blocked?"** — that requires logging on.
8. **Test every change in Count first**, then enforce.

---

## Useful Links

- [AWS WAF Pricing](https://aws.amazon.com/waf/pricing/)
- [AWS WAF Quotas](https://docs.aws.amazon.com/waf/latest/developerguide/limits.html)
- [WCU Reference](https://docs.aws.amazon.com/waf/latest/developerguide/aws-waf-capacity-units.html)
- [WAFv2 API Errors](https://docs.aws.amazon.com/waf/latest/APIReference/CommonErrors.html)
- [Testing & Tuning Rules](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-testing.html)
- [Troubleshooting Blocked Requests](https://docs.aws.amazon.com/waf/latest/developerguide/waf-tips.html)
- [Service Quotas Console](https://console.aws.amazon.com/servicequotas/)

---
