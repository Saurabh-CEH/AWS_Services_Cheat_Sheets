# AWS WAF - Managed Rule Groups Cheat Sheet

## Overview

**Managed rule groups** are pre-built, maintained collections of rules you add to a Web ACL. AWS maintains the **AWS Managed Rules**; third parties sell rule groups through **AWS Marketplace**. They save you from writing and updating signatures yourself.

**Key point:** You reference a managed rule group with a `ManagedRuleGroupStatement`. You control its impact with **OverrideAction** (None or Count) and **rule action overrides** / **scope-down** statements.

---

## Managed Rule Group Types

| Type                        | Description                                                        | Cost              |
| --------------------------- | ------------------------------------------------------------------ | ----------------- |
| **AWS Managed Rules (free tier)** | Baseline & use-case rule sets maintained by AWS            | No extra rule fee |
| **AWS Managed Rules (intelligent, paid)** | Bot Control, ATP, ACFP — usage-based pricing      | Extra fee         |
| **AWS Marketplace**         | Third-party vendor rule groups (subscription)                      | Vendor pricing    |

---

## Common AWS Managed Rule Groups

| Rule group                              | Purpose                                                     | Approx WCU |
| --------------------------------------- | ----------------------------------------------------------- | ---------- |
| **AWSManagedRulesCommonRuleSet (CRS)**  | Broad OWASP-style protections (core baseline)               | 700        |
| **AWSManagedRulesKnownBadInputsRuleSet**| Blocks request patterns known to be invalid/exploit         | 200        |
| **AWSManagedRulesSQLiRuleSet**          | SQL injection protections                                   | 200        |
| **AWSManagedRulesLinuxRuleSet**         | Linux-specific LFI/exploit patterns                         | 200        |
| **AWSManagedRulesUnixRuleSet (POSIX)**  | POSIX/Unix command injection                                | 100        |
| **AWSManagedRulesWindowsRuleSet**       | Windows/PowerShell exploit patterns                         | 200        |
| **AWSManagedRulesPHPRuleSet**           | PHP application exploits                                     | 100        |
| **AWSManagedRulesWordPressRuleSet**     | WordPress-specific protections                              | 100        |
| **AWSManagedRulesAmazonIpReputationList** | AWS threat-intel IP reputation + DDoS/reconnaissance lists | 25         |
| **AWSManagedRulesAnonymousIpList**      | Blocks VPNs, Tor, hosting-provider/anonymizer IPs           | 50         |
| **AWSManagedRulesBotControlRuleSet**    | Bot detection & categorization (paid — see Bot Control)     | 50         |
| **AWSManagedRulesATPRuleSet**           | Account Takeover Prevention (paid)                          | 50         |
| **AWSManagedRulesACFPRuleSet**          | Account Creation Fraud Prevention (paid)                    | 50         |
| **AWSManagedRulesAntiDDoSRuleSet**      | Application-layer DDoS detection/mitigation (paid — challenge + block) | 50 |

> WCU numbers are approximate and can change — always check `describe-managed-rule-group` for current capacity before budgeting.

---

## AWSManagedRulesCommonRuleSet — Notable Rules

| Rule name                    | Blocks                                                  |
| ---------------------------- | ------------------------------------------------------- |
| `SizeRestrictions_BODY`      | Bodies over ~8 KB (common false-positive source)       |
| `SizeRestrictions_QUERYSTRING` | Oversized query strings                               |
| `NoUserAgent_HEADER`         | Requests with missing User-Agent                       |
| `CrossSiteScripting_BODY`    | XSS in request body                                    |
| `GenericLFI_URIPATH`         | Local file inclusion patterns                          |
| `EC2MetaDataSSRF_BODY`       | SSRF attempts to EC2 metadata endpoint                 |

> `SizeRestrictions_BODY` frequently blocks legitimate file uploads. If you see 403s on uploads, this is the usual culprit — override it to Count or exclude it.

---

## AWSManagedRulesAntiDDoSRuleSet (Anti-DDoS) — Intelligent Threat Mitigation

An **intelligent-threat-mitigation** managed rule group (paid, WCU **50**) that detects and manages requests **participating in application-layer DDoS attacks**. It **labels every request** to a protected resource during a probable event and applies soft/hard mitigations you can tune.

**Two mitigation types, each tunable to Low / Medium / High suspicion:**
- **Soft mitigation** — sends **silent browser Challenges** to requests that can handle the challenge interstitial.
- **Hard mitigation** — **Blocks** requests outright.

### Rules (evaluated in order)

| Rule                    | Action     | Notes                                                                                     |
| ----------------------- | ---------- | ----------------------------------------------------------------------------------------- |
| `ChallengeAllDuringEvent` | **Challenge** | Challenges **all** `challengeable-request`-labeled traffic while the resource is under attack. Overridable **only to Allow or Count** (Allow not recommended). |
| `ChallengeDDoSRequests`   | **Challenge** | Challenges requests meeting the configured **challenge sensitivity**. **Only evaluated if you set `ChallengeAllDuringEvent` to Count.** |
| `DDoSRequests`            | **Block**  | Blocks requests meeting the configured **block sensitivity** during an attack.            |

### Key labels

| Label                                                       | Meaning                                                        |
| ----------------------------------------------------------- | -------------------------------------------------------------- |
| `awswaf:managed:aws:anti-ddos:event-detected`               | Resource is in a detected DDoS event (added to **all** traffic — legit + attack) |
| `awswaf:managed:aws:anti-ddos:ddos-request`                 | Request is from a source suspected of participating            |
| `...:low-/medium-/high-suspicion-ddos-request`              | Confidence level of participation                              |
| `awswaf:managed:aws:anti-ddos:challengeable-request`        | URI can take a silent challenge (not on the exempt-URI regex list) |

- Because Challenge is used, the same **token state labels** apply (`token:accepted`, `token:rejected:*`, `token:absent`, `captcha:*`) — see the CAPTCHA & Challenge sheet.
- The group **labels but doesn't always act** — you can add a **label-match rule after it** for custom handling.
- **Relationship to Shield Advanced:** this is the WAF-side, application-layer complement to Shield Advanced's automatic L7 mitigation; AWS is migrating some Shield Advanced L7 behavior onto this rule group.

### Anti-DDoS gotchas

1. **Paid + Challenge-based** — adds WAF fees *and* CAPTCHA/Challenge fees when it challenges; monitor cost during events.
2. **`event-detected` tags legitimate traffic too** — don't treat that label alone as "this request is an attacker."
3. **`ChallengeDDoSRequests` is dormant unless you set `ChallengeAllDuringEvent` to Count** — a common configuration surprise.
4. **Challenge needs JS/HTML-capable clients** — pure API/non-browser clients can't solve challenges; scope-down or exempt those URIs.
5. **Requires token handling** to be effective (SDK or the challenge interstitial); token-domain misconfig makes requests look token-absent.

---

## Controlling a Managed Rule Group

### 1. OverrideAction (whole group)

| OverrideAction | Effect                                                          |
| -------------- | -------------------------------------------------------------- |
| **None**       | Use each rule's own action (normal enforcement)                |
| **Count**      | Force **every** rule in the group to Count (safe testing mode) |

### 2. Rule action overrides (per rule inside the group)

Override individual rules without touching the rest:

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesCommonRuleSet",
    "RuleActionOverrides": [
      { "Name": "SizeRestrictions_BODY", "ActionToUse": { "Count": {} } },
      { "Name": "NoUserAgent_HEADER",   "ActionToUse": { "Allow": {} } }
    ]
  }
}
```

### 3. Scope-down statement (limit what the group inspects)

Only apply the managed group to a subset of traffic:

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesBotControlRuleSet",
    "ManagedRuleGroupConfigs": [
      { "AWSManagedRulesBotControlRuleSet": { "InspectionLevel": "COMMON" } }
    ],
    "ScopeDownStatement": {
      "ByteMatchStatement": {
        "SearchString": "/api/",
        "FieldToMatch": { "UriPath": {} },
        "PositionalConstraint": "STARTS_WITH",
        "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
      }
    }
  }
}
```

### 4. Version pinning

- Some groups support **versioning**. You can pin a `Version` or use the default (latest).
- AWS publishes new versions; a `Version_X.X` locks behavior. Test new versions in Count before adopting.

---

## Excluding vs Overriding (terminology)

| Old term (Classic-style) | Current WAFv2 mechanism                    |
| ------------------------ | ------------------------------------------ |
| "Excluded rules"         | `RuleActionOverrides` set to **Count**     |
| Disable a rule           | Override to Count (still logged) or Allow  |

> Setting a rule override to **Count** is the standard way to "exclude" a noisy rule while keeping visibility.

---

## CLI Commands

```bash
# List all managed rule groups available (AWS + Marketplace)
aws wafv2 list-available-managed-rule-groups \
  --scope REGIONAL --region us-east-1

# Describe a managed rule group (rule names, labels, WCU)
aws wafv2 describe-managed-rule-group \
  --vendor-name AWS \
  --name AWSManagedRulesCommonRuleSet \
  --scope REGIONAL \
  --region us-east-1

# List versions of a managed rule group
aws wafv2 list-available-managed-rule-group-versions \
  --vendor-name AWS \
  --name AWSManagedRulesCommonRuleSet \
  --scope REGIONAL \
  --region us-east-1
```

Add a managed group to a Web ACL rule (Count override for testing):

```json
{
  "Name": "AWS-CommonRuleSet",
  "Priority": 1,
  "Statement": {
    "ManagedRuleGroupStatement": {
      "VendorName": "AWS",
      "Name": "AWSManagedRulesCommonRuleSet"
    }
  },
  "OverrideAction": { "Count": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "AWSCommonRuleSet"
  }
}
```

---

## Recommended Baseline Stack

```
Priority 0:  AmazonIpReputationList         (None)
Priority 1:  AnonymousIpList                (None — or Count if you have legit VPN users)
Priority 2:  CommonRuleSet                  (None, override SizeRestrictions_BODY to Count)
Priority 3:  KnownBadInputsRuleSet          (None)
Priority 4:  SQLiRuleSet                    (None)
Priority 5:  Custom rate-based rule         (Block)
Default action: ALLOW
```

Add platform-specific groups (WordPress/PHP/Linux/Windows) only if they match your stack — they add WCU and false-positive surface otherwise.

---

## Gotchas & Caveats

1. **`SizeRestrictions_BODY` (in CRS) is the #1 false-positive** — it blocks bodies over ~8 KB, breaking file/Excel/multipart uploads. Override it to Count or raise the body limit.
2. **`NoUserAgent_HEADER` blocks legit no-UA clients** — health checkers, some API clients, and IoT devices. Override if needed.
3. **CommonRuleSet is ~700 WCU** — nearly half the default 1,500 budget. Adding several groups can exceed the cap; plan or request an increase.
4. **You cannot delete individual rules from a managed group** — you can only override their action (to Count/Allow). There is no "remove rule."
5. **Auto-adopting the latest version can break traffic** — new versions change signatures. Pin a `Version` in production; test upgrades in Count.
6. **Overriding to Count still logs and labels** — it does not disable the rule's evaluation, just its blocking. Use it to "exclude" a rule while keeping visibility.
7. **Platform packs add false-positive surface** — don't add WordPress/PHP/Windows/Linux groups unless you actually run that stack.
8. **`AnonymousIpList` blocks VPN/Tor/hosting IPs** — if you have legitimate VPN users or cloud-hosted integrations, this causes false blocks.
9. **`AmazonIpReputationList` updates on AWS's schedule** — it won't reflect a brand-new attacker IP instantly; combine with rate-based rules.
10. **Marketplace rule groups require an active subscription** — an expired/cancelled subscription disables the group (and can surprise you in FMS deployments).
11. **Scope-down applies to the whole group** — it decides *whether* the group runs, not which internal rules; use RuleActionOverrides for per-rule control.
12. **FMS-deployed managed groups can appear "missing" in member accounts** — that's usually a policy scope/remediation issue, not the account's Web ACL.

### Official Caveats (from AWS docs)

1. **AWS publishes limited rule detail on purpose.** AWS documents enough to use the rules without giving attackers what they need to evade them. For more detail, use `DescribeManagedRuleGroup` or contact AWS Support.
2. **Docs describe the latest static version.** A group's documented behavior applies to its most recent static version; other versions can differ. Track changes via the AWS Managed Rules changelog and pin a version for stability.
3. **Intelligent-threat groups carry extra fees.** Bot Control, ATP, and ACFP are billed additional fees on top of base WAF charges — follow the intelligent-threat-mitigation best practices to control cost.
4. **`SizeRestrictions_BODY` reflects the body inspection limit**, which is 8 KB on ALB/AppSync and 16 KB (default, up to 64 KB) on CloudFront/API GW/Cognito/App Runner — this is why oversized uploads trip it.
5. **Fraud groups (ATP/ACFP) are not available for Cognito user pools** — you can't add them to, or associate their Web ACL with, a Cognito user pool.
6. **Each managed group publishes a fixed WCU cost** — retrieve current values with `DescribeManagedRuleGroup` before budgeting; the numbers in this sheet are approximate.

> Source: [AWS Managed Rules rule groups list](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html) and the individual rule-group pages. Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                          | Cause                                           | Fix                                                          |
| ---------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| File uploads return 403                        | `SizeRestrictions_BODY` in CRS                  | Override that rule to Count; raise body inspection limit     |
| Legit users blocked (missing UA / VPN)         | `NoUserAgent_HEADER` or AnonymousIpList         | Override specific rule to Count/Allow                        |
| Group not blocking anything                    | OverrideAction set to Count                      | Set OverrideAction to None to enforce                        |
| WCU limit exceeded after adding group          | CRS (~700 WCU) plus others                      | Remove unneeded groups or request WCU increase               |
| New group version broke traffic                | Auto-adopted latest version                      | Pin a known-good `Version`, test new ones in Count           |
| Managed rule not identifiable in logs          | Sampling/labels not enabled                      | Enable sampled requests; inspect labels in logs              |

---

## Best Practices

1. **Onboard every managed group in Count mode first**, then enforce.
2. **Start with reputation + CRS + KnownBadInputs** — the highest-value baseline.
3. **Only add platform packs you actually run** (WordPress/PHP/etc.).
4. **Override noisy rules to Count** rather than removing whole groups.
5. **Watch your WCU budget** — CRS alone is ~700 of 1,500 default.
6. **Pin versions in production** and test upgrades in a staging Web ACL.
7. **Use scope-down statements** to focus expensive groups (Bot Control) on specific paths.
8. **Review AWS change notifications** — managed rules update over time.

---

## Useful Links

- [AWS Managed Rule Groups List](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html)
- [Baseline Managed Rule Groups](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-baseline.html)
- [Use-case Specific Rule Groups](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-use-case.html)
- [IP Reputation Rule Groups](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-ip-rep.html)
- [Anti-DDoS (DDoS prevention) rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-anti-ddos.html)
- [Advanced Anti-DDoS protection (configuration)](https://docs.aws.amazon.com/waf/latest/developerguide/waf-anti-ddos-advanced.html)
- [Overriding Rule Actions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-group-override-options.html)
- [Managed Rule Group Versioning](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-rule-groups-versioning.html)
- [AWS Marketplace Managed Rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-rule-groups-marketplace.html)

---
