# AWS WAF - Overview & Core Concepts Cheat Sheet

## Overview

AWS WAF (Web Application Firewall) is a Layer 7 firewall that protects web applications and APIs from common web exploits and bots. It inspects incoming HTTP(S) requests against rules you define and either allows, blocks, counts, or challenges them.

**Key point:** This cheat sheet series covers **AWS WAF (WAFv2)** — the current version. Legacy "WAF Classic" is a separate, older API (see the Migration cheat sheet).

---

## What WAF Protects Against

| Threat                          | Description                                                        |
| ------------------------------- | ------------------------------------------------------------------ |
| **SQL injection (SQLi)**        | Malicious SQL in request parameters                                |
| **Cross-site scripting (XSS)**  | Injected client-side scripts                                       |
| **Bad bots / scrapers**         | Automated traffic, credential stuffing, scraping                   |
| **DDoS (Layer 7)**              | Application-layer floods (rate-based rules + Shield)               |
| **Known-bad IPs**               | Reputation lists, anonymizers, hosting-provider IPs                |
| **Account takeover (ATO)**      | Credential stuffing against login endpoints                        |
| **Account creation fraud**      | Automated fake sign-ups                                            |

---

## Core Concepts

| Component               | Description                                                                  |
| ----------------------- | ---------------------------------------------------------------------------- |
| **Web ACL**             | Top-level container of rules; associated with protected resources            |
| **Rule**                | A statement + an action (Allow / Block / Count / CAPTCHA / Challenge)        |
| **Rule Statement**      | The match condition (what to inspect and how)                                |
| **Rule Group**          | Reusable collection of rules (managed by AWS/vendor or self-managed)         |
| **Managed Rule Group**  | Pre-built rules maintained by AWS or Marketplace sellers                     |
| **IP Set**              | A reusable list of IP addresses / CIDR ranges                                |
| **Regex Pattern Set**   | A reusable list of regular expressions                                       |
| **Web ACL Capacity Unit (WCU)** | Cost/complexity unit measuring how much processing a rule needs      |
| **Scope**               | `REGIONAL` (ALB, API GW, AppSync, etc.) or `CLOUDFRONT` (global)             |
| **Default action**      | What happens when no rule matches (Allow or Block)                           |

---

## Rule Actions

| Action        | Behavior                                                                 |
| ------------- | ------------------------------------------------------------------------ |
| **Allow**     | Permit the request. Stop evaluating rules in this Web ACL.               |
| **Block**     | Deny the request. Returns 403 by default (customizable response).       |
| **Count**     | Increment a counter and continue evaluating. Used for testing/tuning.   |
| **CAPTCHA**   | Return a CAPTCHA puzzle; passes if solved. Good for interactive clients. |
| **Challenge** | Silent browser challenge (JS/proof-of-work); no user interaction.       |

> **Count** does not stop evaluation — it is for observability. **Allow/Block** are terminating actions. **CAPTCHA/Challenge** are terminating only if the client fails.

---

## Supported Resources (Associations)

| Resource                          | Scope       |
| --------------------------------- | ----------- |
| **Amazon CloudFront**             | CLOUDFRONT (global, us-east-1) |
| **Application Load Balancer (ALB)** | REGIONAL  |
| **Amazon API Gateway (REST)**     | REGIONAL    |
| **AWS AppSync (GraphQL)**         | REGIONAL    |
| **Amazon Cognito user pool**      | REGIONAL    |
| **AWS App Runner service**        | REGIONAL    |
| **Verified Access instance**      | REGIONAL    |
| **AWS Amplify**                   | REGIONAL    |

> **Not supported:** Network Load Balancer (Layer 4), Classic Load Balancer (use WAF Classic or migrate), EC2 directly. Put an ALB or CloudFront in front.

---

## How WAF Evaluates a Request

```
Incoming HTTP(S) request
        |
        v
Web ACL rules evaluated IN PRIORITY ORDER (lowest number first)
        |
        ├── Rule matches with terminating action (Allow/Block) → STOP, apply action
        |
        ├── Rule matches with Count → log/count, CONTINUE to next rule
        |
        ├── CAPTCHA/Challenge → client challenged; pass = continue, fail = terminate
        |
        └── No rule matches → apply Web ACL DEFAULT ACTION (Allow or Block)
```

### Key Evaluation Rules

1. Rules are processed in **priority order** — lowest priority number first.
2. First **terminating** action wins (Allow or Block).
3. **Count** actions never stop evaluation.
4. Inside a rule group, the group's rules run before moving to the next top-level rule.
5. If nothing matches, the **default action** decides the outcome.

---

## Regional vs CloudFront Scope

| Aspect             | REGIONAL                                  | CLOUDFRONT                          |
| ------------------ | ----------------------------------------- | ----------------------------------- |
| Resources          | ALB, API GW, AppSync, Cognito, App Runner | CloudFront distributions            |
| Region for API     | The region of the resource                | Must use **us-east-1**              |
| Reach              | Single region                             | Global (edge)                       |
| CLI `--scope`      | `REGIONAL`                                | `CLOUDFRONT`                        |

> To manage a CloudFront Web ACL via CLI, you must target `--region us-east-1 --scope CLOUDFRONT`.

---

## WAF vs Shield vs Firewall Manager

| Service                | Purpose                                                                 |
| ---------------------- | ----------------------------------------------------------------------- |
| **AWS WAF**            | Layer 7 request filtering (rules you define)                            |
| **AWS Shield Standard**| Free, automatic Layer 3/4 DDoS protection for all AWS customers         |
| **AWS Shield Advanced**| Paid; enhanced DDoS protection, DDoS cost protection, SRT, WAF included |
| **AWS Firewall Manager**| Centrally deploy & enforce WAF/Shield policies across an Organization  |

> These are complementary. Common pattern: Shield Advanced + WAF managed rules + rate-based rules, all deployed via Firewall Manager.

---

## Quick-Start CLI

```bash
# List Web ACLs (REGIONAL)
aws wafv2 list-web-acls --scope REGIONAL --region us-east-1

# List Web ACLs (CLOUDFRONT — must be us-east-1)
aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1

# Get details of a Web ACL
aws wafv2 get-web-acl \
  --name my-web-acl \
  --scope REGIONAL \
  --id 12345678-1234-1234-1234-123456789012 \
  --region us-east-1

# List available AWS managed rule groups
aws wafv2 list-available-managed-rule-groups \
  --scope REGIONAL \
  --region us-east-1

# Associate a Web ACL with an ALB
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc123" \
  --resource-arn "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/abc" \
  --region us-east-1

# List resources protected by a Web ACL
aws wafv2 list-resources-for-web-acl \
  --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-web-acl/abc123" \
  --region us-east-1
```

---

## Web ACL Capacity Units (WCU)

- Every rule consumes **WCUs** based on its complexity.
- A Web ACL has a **default maximum of 1,500 WCUs** (increasable via quota request).
- Managed rule groups publish their WCU cost — budget for it when combining many.
- Simple statements (e.g., IP match) cost little; regex, bot control, and nested logic cost more.

| Example statement            | Approx WCU |
| ---------------------------- | ---------- |
| IP set match                 | 1          |
| Geo match                    | 1          |
| Byte match (string)          | 1 (+ transformations) |
| Regex pattern set            | 25 (base)  |
| SQLi / XSS match             | ~10 – 20   |
| Rate-based rule              | 2          |
| Managed rule groups          | 10 – 700+ (varies) |

---

## Cheat Sheet Series Map

| Topic                              | See |
| ---------------------------------- | --- |
| Web ACLs & rule structure          | Web ACLs & Rules |
| Match conditions & transformations | Rule Statements & Match Conditions |
| AWS/vendor managed rules           | Managed Rule Groups |
| Rate limiting & L7 DDoS            | Rate-Based Rules |
| Bot mitigation, ATO, ACFP          | Bot Control |
| Reusable IP / regex sets           | IP Sets & Regex Pattern Sets |
| Logging, metrics, sampled requests | Logging & Monitoring |
| Classic → WAFv2 migration          | Migration |
| Cost, limits, common errors        | Pricing, Quotas & Troubleshooting |

---

## Gotchas & Caveats

1. **CloudFront Web ACLs must live in `us-east-1`** — with `--scope CLOUDFRONT`. Managing them from any other region fails or shows nothing.
2. **One Web ACL per resource** — you cannot stack two Web ACLs on the same ALB/CloudFront/API. Combine rules into one Web ACL instead.
3. **NLB and Classic Load Balancer are not supported** — WAF is Layer 7 only. Front them with an ALB or CloudFront.
4. **Default WCU cap is 1,500** — `AWSManagedRulesCommonRuleSet` alone is ~700. You can exhaust the budget fast; request an increase before stacking groups.
5. **Count never terminates evaluation** — a Count match does not stop later rules. Only Allow/Block (and failed CAPTCHA/Challenge) are terminating.
6. **Association is not instant** — associating/disassociating a Web ACL (especially CloudFront) can take a few minutes to propagate globally.
7. **WAF sees the edge/proxy IP, not the client**, when behind CloudFront or another proxy — use forwarded-IP handling for IP/geo/rate logic.
8. **Region matters for REGIONAL scope** — the Web ACL must be in the same region as the ALB/API Gateway it protects.
9. **Body inspection is limited by default** (ALB/AppSync 8 KB; CloudFront/API GW/Cognito/App Runner 16 KB default, up to 64 KB) — large payloads are only partially inspected unless you raise the limit and set oversize handling.
10. **WAF is request filtering, not DDoS mitigation on its own** — pair with Shield (and rate-based rules) for volumetric attacks.

---

## Best Practices

1. **Start rules in Count mode** — Observe matches before switching to Block to avoid breaking legit traffic.
2. **Layer defenses** — Combine AWS managed rules + rate-based rules + custom rules.
3. **Set a sensible default action** — Allow-by-default for most sites; Block-by-default for locked-down APIs (allowlist pattern).
4. **Enable logging early** — You cannot tune what you cannot see.
5. **Mind the WCU budget** — Plan capacity before stacking many managed rule groups.
6. **Use Firewall Manager** for multi-account governance.
7. **Pair with Shield Advanced** for DDoS-sensitive workloads.
8. **Version-control your rules** — Manage Web ACLs as IaC (CloudFormation/Terraform) for repeatability.

---

## Useful Links

- [What is AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html)
- [How AWS WAF Works](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html)
- [Web ACLs](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl.html)
- [Rule Actions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-action.html)
- [WCU Reference](https://docs.aws.amazon.com/waf/latest/developerguide/aws-waf-capacity-units.html)
- [Supported Resources](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html)
- [AWS WAF Pricing](https://aws.amazon.com/waf/pricing/)

---
