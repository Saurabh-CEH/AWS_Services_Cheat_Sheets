# AWS WAF - Web ACLs & Rules Cheat Sheet

## Overview

A **Web ACL (Web Access Control List)** is the top-level WAF resource. It holds an ordered set of rules, a default action, and is associated with one or more protected resources (CloudFront, ALB, API Gateway, etc.). Rules define what to inspect and what to do on a match.

**Key point:** A resource can be associated with **only one Web ACL** at a time, but one Web ACL can protect **many resources**.

---

## Web ACL Anatomy

| Element               | Description                                                              |
| --------------------- | ------------------------------------------------------------------------ |
| **Name / ID / ARN**   | Identifiers. ID + name required for most CLI calls                       |
| **Scope**             | `REGIONAL` or `CLOUDFRONT`                                               |
| **Default action**    | Allow or Block when no rule produces a terminating action               |
| **Rules**             | Ordered list evaluated by priority                                       |
| **Rule labels**       | Metadata added on match; later rules can match on labels                 |
| **Custom responses**  | Custom body/status returned on Block                                     |
| **CloudWatch metric** | Aggregate allowed/blocked counts                                         |
| **WCU usage**         | Sum of all rule WCUs (default cap 1,500)                                |
| **Association config**| Body inspection size limits per resource type                            |

---

## Rule Structure

Each rule contains:

```
Rule
 ├── Name              (unique within the Web ACL)
 ├── Priority          (integer; lower = evaluated first)
 ├── Statement         (the match condition — see Match Conditions sheet)
 ├── Action            (Allow / Block / Count / CAPTCHA / Challenge)
 │     └── OR OverrideAction (only for rule-group reference statements)
 ├── VisibilityConfig  (CloudWatch metric name, sampled requests toggle)
 └── RuleLabels        (optional labels applied on match)
```

### Action vs OverrideAction

| Field              | Used for                        | Values                                        |
| ------------------ | ------------------------------- | --------------------------------------------- |
| **Action**         | Regular rules (your statements) | Allow, Block, Count, CAPTCHA, Challenge       |
| **OverrideAction** | Rule-group reference statements | None (use group's actions) or Count (test)    |

> You cannot set `Action` on a rule that references a rule group — you set `OverrideAction` instead. Use `Count` override to test a managed group without blocking.

---

## Default Action Patterns

| Pattern              | Default action | Use case                                          |
| -------------------- | -------------- | ------------------------------------------------- |
| **Blocklist**        | Allow          | Public sites — allow everyone, block known bad    |
| **Allowlist**        | Block          | Internal APIs — block everyone, allow known good  |

### Blocklist example (default Allow)

```
Priority 0:  Block  → AWS managed common rule set
Priority 1:  Block  → SQLi rule set
Priority 2:  Block  → Rate-based rule (>2000 req / 5 min)
Default action: ALLOW
```

### Allowlist example (default Block)

```
Priority 0:  Allow  → IP set (corporate egress IPs)
Priority 1:  Allow  → Requests with valid API key header
Default action: BLOCK
```

---

## How Priority Works

```
Requests enter
      |
      v
Priority 0  → 1 → 2 → 3 ... (ascending)
      |
      ├── terminating match (Allow/Block) → STOP
      ├── Count match → continue
      └── no match anywhere → DEFAULT ACTION
```

- Priorities need not be contiguous (0, 10, 20 is fine) — only relative order matters.
- Put **allow** rules for trusted traffic **before** broad block rules.
- Put **specific** rules before **general** ones.

---

## Rule Labels

Labels let rules communicate. A rule (or managed rule group) adds a label on match; a later rule matches on that label.

```
Priority 0:  AWS managed BotControl → adds label awswaf:managed:aws:bot-control:bot:category:search_engine
Priority 1:  Allow → LabelMatchStatement on "search_engine" label
Priority 2:  Block → LabelMatchStatement on "awswaf:managed:aws:bot-control:bot:verified:false"
```

| Label source          | Example                                                    |
| --------------------- | ---------------------------------------------------------- |
| Managed rule groups   | `awswaf:managed:aws:bot-control:bot:category:...`          |
| Your own rules        | `myorg:custom:label`                                        |

> Labels are evaluated within the **same Web ACL** and only visible to rules with a **higher priority number** (evaluated later).

---

## Custom Responses & Request Headers

| Feature                    | Description                                                      |
| -------------------------- | --------------------------------------------------------------- |
| **Custom response (Block)**| Return custom status code (e.g., 429) and body instead of 403   |
| **Custom request header**  | Insert `x-amzn-waf-*` headers passed to your origin on Allow    |
| **Response bodies**        | Defined at Web ACL level, referenced by rules                   |

```bash
# Custom response bodies are defined in the Web ACL's customResponseBodies map,
# then referenced by a rule's Block action -> CustomResponse.
```

---

## CLI: Creating & Managing Web ACLs

Because Web ACL JSON is large, most teams pass a file. Example workflow:

```bash
# 1. Create a Web ACL with default Allow and one managed rule group (Count mode)
aws wafv2 create-web-acl \
  --name my-web-acl \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --visibility-config \
      SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myWebAcl \
  --rules '[
    {
      "Name": "AWSCommonRules",
      "Priority": 0,
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
        "MetricName": "AWSCommonRules"
      }
    }
  ]'

# 2. Get the Web ACL (note the LockToken returned — required for updates)
aws wafv2 get-web-acl \
  --name my-web-acl --scope REGIONAL \
  --id 12345678-1234-1234-1234-123456789012 \
  --region us-east-1

# 3. Update the Web ACL (must supply the current --lock-token)
aws wafv2 update-web-acl \
  --name my-web-acl --scope REGIONAL \
  --id 12345678-1234-1234-1234-123456789012 \
  --lock-token abcd1234-... \
  --default-action Allow={} \
  --visibility-config \
      SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myWebAcl \
  --rules file://rules.json \
  --region us-east-1

# 4. Delete a Web ACL (must be disassociated first; needs lock token)
aws wafv2 delete-web-acl \
  --name my-web-acl --scope REGIONAL \
  --id 12345678-1234-1234-1234-123456789012 \
  --lock-token abcd1234-... \
  --region us-east-1
```

### The LockToken

- Every `get-*` returns a **LockToken**.
- `update-*` and `delete-*` require the **current** lock token (optimistic locking).
- If someone else changed the resource, your token is stale → `WAFOptimisticLockException`. Re-`get` and retry.

---

## Associating & Disassociating Resources

```bash
# Associate (ALB / API GW / AppSync / Cognito / App Runner)
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:...:loadbalancer/app/my-alb/xyz" \
  --region us-east-1

# Disassociate
aws wafv2 disassociate-web-acl \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:...:loadbalancer/app/my-alb/xyz" \
  --region us-east-1

# What Web ACL protects a resource?
aws wafv2 get-web-acl-for-resource \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:...:loadbalancer/app/my-alb/xyz" \
  --region us-east-1
```

> **CloudFront:** you don't call `associate-web-acl`. Instead set the Web ACL ARN in the CloudFront distribution config (`WebACLId`), and manage the Web ACL in `us-east-1 / CLOUDFRONT`.

---

## Testing Rules Safely (Count Mode)

1. Add the rule with **Action = Count** (or **OverrideAction = Count** for rule groups).
2. Enable logging + sampled requests.
3. Watch CloudWatch `CountedRequests` and sampled requests for false positives.
4. Once confident, flip to **Block**.

---

## Body Inspection Size Limits

WAF inspects only part of the request body by default. You can raise it via association config (`AssociationConfig` → `RequestBody`).

| Resource type    | Default inspected | Max (with oversize handling) |
| ---------------- | ----------------- | ---------------------------- |
| CloudFront, API Gateway, Cognito, App Runner, Verified Access, Bedrock AgentCore | 16 KB | up to 64 KB |
| ALB, AppSync     | 8 KB              | (fixed — not raisable)       |

> Beyond the limit, use the `OversizeHandling` option (CONTINUE / MATCH / NO_MATCH) on body statements. Larger inspection may increase WCU cost.

---

## Gotchas & Caveats

1. **You cannot set `Action` on a rule-group reference** — use `OverrideAction` (None or Count). Setting Action is a common validation error.
2. **`OverrideAction: Count` overrides the whole group** — every rule inside becomes Count. To silence just one noisy rule, use per-rule `RuleActionOverrides` instead.
3. **Lock tokens are single-use per version** — every update/delete needs the current `LockToken` from a fresh `get-*`. Stale token → `WAFOptimisticLockException`.
4. **You must disassociate all resources before deleting a Web ACL** — otherwise `WAFUnavailableEntityException`.
5. **CloudFront is not associated via `associate-web-acl`** — you set the Web ACL ARN in the distribution config; managing it requires `us-east-1 / CLOUDFRONT`.
6. **Labels are only visible to later rules** — a rule can only match a label added by a rule with a **lower** priority number. Order matters.
7. **Priorities must be unique** — gaps are fine (0/10/20), ties are rejected; only relative order matters.
8. **Default action fires only when nothing terminates** — a stray high-priority Allow can silently short-circuit all your block rules.
9. **Custom response bodies are defined at the Web ACL level** — a rule references them by key; you can't inline an arbitrary body per rule.
10. **Body inspection size is set via `AssociationConfig`** and larger inspection raises WCU cost — it is not free.
11. **Continuous Deployment (CloudFront) can complicate WAF association** — verify the Web ACL is attached to the correct (staging vs primary) distribution config.

---

## Troubleshooting

| Issue                                   | Cause                                             | Fix                                                             |
| --------------------------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| `WAFOptimisticLockException` on update  | Stale lock token                                  | Re-run `get-web-acl`, use the fresh `LockToken`                |
| `WAFUnavailableEntityException`         | Resource still associated when deleting Web ACL   | Disassociate all resources first                               |
| Rule never matches                      | Higher-priority terminating rule fires first      | Reorder priorities; check for earlier Allow                    |
| Managed group won't set Action          | You set `Action` instead of `OverrideAction`      | Use `OverrideAction` for rule-group references                 |
| Legit traffic blocked after adding rule | Rule too broad / false positive                   | Switch rule to Count, inspect sampled requests, add exclusions |
| Cannot associate with NLB/CLB           | Unsupported resource type                          | Front it with an ALB or CloudFront                             |
| WCU limit exceeded                      | Too many/complex rules                             | Trim rules or request WCU quota increase                       |

---

## Best Practices

1. **One Web ACL, many resources** — reuse a Web ACL across similar apps for consistency.
2. **Deploy new rules in Count first** — validate before blocking.
3. **Order matters** — allow trusted traffic early, block broad threats after specifics.
4. **Use labels** to build multi-stage logic instead of duplicating conditions.
5. **Keep a WCU budget** — track capacity as you add managed groups.
6. **Automate with IaC** — version-control Web ACL definitions.
7. **Custom 429 responses** for rate limits give clients a clearer signal than 403.
8. **Always cache the lock token** in automation and refresh on conflict.

---

## Useful Links

- [Web ACLs](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl.html)
- [Working with Rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rules.html)
- [Rule Action](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-action.html)
- [Rule Priority](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-processing.html)
- [Labels on Web Requests](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-label-match-statement.html)
- [Custom Responses](https://docs.aws.amazon.com/waf/latest/developerguide/customizing-the-response-for-blocked-requests.html)
- [Associating a Web ACL](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-associating-aws-resource.html)
- [Body Inspection & Oversize Handling](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-fields.html)

---
