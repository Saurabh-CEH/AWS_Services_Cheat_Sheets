# AWS Firewall Manager - Shield Advanced Policy Cheat Sheet

## Overview

An **FMS Shield Advanced policy** centrally enrolls in-scope resources into **Shield Advanced protection** across an Organization, so you don't manually create protections in each account. It ensures existing and future resources of the chosen types get Shield Advanced coverage.

**Key point:** The FMS Shield policy still requires an **active Shield Advanced subscription** at the org level — FMS automates *enrollment*, not the subscription.

---

## What It Manages

| Capability                       | Detail                                                        |
| -------------------------------- | ------------------------------------------------------------- |
| **Automatic protection**         | Enrolls in-scope resources (CloudFront, ALB, NLB, EIP, etc.) into Shield Advanced |
| **Scope**                        | By account/OU, resource type, and tags                        |
| **Auto-remediation**             | Protects new/non-compliant resources automatically            |
| **Compliance reporting**         | Which resources are/aren't protected                          |

---

## How It Works

```
Org has an ACTIVE Shield Advanced subscription
        |
        v
FMS admin defines a Shield policy (resource types + scope + remediation)
        |
        v
FMS creates Shield PROTECTIONS on in-scope resources (existing + future)
        |
        v
Compliance dashboard shows protected vs unprotected resources
```

---

## CLI

```bash
# List / inspect (in FMS admin account)
aws fms list-policies
aws fms get-policy --policy-id <shield-policy-id>
aws fms list-compliance-status --policy-id <shield-policy-id>

# Put a Shield policy — SecurityServicePolicyData.Type = "SHIELD_ADVANCED"
aws fms put-policy --policy file://shield-policy.json
```

---

## Gotchas & Caveats

1. **Requires an active Shield Advanced subscription** — FMS enrolls resources but doesn't buy the subscription; without it, the policy can't protect.
2. **Enrollment ≠ full protection tuning** — FMS creates protections, but health-check association, automatic app-layer mitigation, and SRT setup are still separate steps.
3. **Only Shield-supported resource types** are enrolled (CloudFront, ALB, NLB, CLB, EIP, Global Accelerator, R53) — others are skipped.
4. **Auto-remediation must be ON** to actually enroll resources; otherwise it only reports.
5. **AWS Config required** in scoped accounts/Regions for evaluation.
6. **Regional vs global** — protect global (CloudFront/GA) and regional resources per their scope; plan policies accordingly.
7. **Cost awareness** — enrolling many resources doesn't add per-resource Shield fees (subscription is org-level), but **data transfer out** and health-check costs still apply.
8. **De-scoping behavior** — removing a resource from scope may leave or drop its protection; verify.
9. **Health-based detection** still needs Route 53 health checks associated (not done by the enrollment policy itself).
10. **Throttling at scale** possible during evaluation.
11. **Cross-account** — protections are created in resource-owning accounts under the org subscription.
12. **FMS Shield policy complements, not replaces, WAF** — L7 defense still needs a WAF policy/Web ACL association.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Resources not getting protected             | No subscription / Config off / remediation off  | Ensure Shield Adv subscription; enable Config + remediation |
| Some resources skipped                      | Unsupported resource type                       | Only Shield-supported types are enrolled                  |
| Protected but noisy detection               | No health checks associated                     | Associate Route 53 health checks (separate step)          |
| L7 attacks still hit apps                   | No WAF policy / Web ACL                          | Add an FMS WAF policy / associate WAF                     |
| Compliance shows non-compliant              | Config lag / evaluation pending                  | Verify Config recording; wait for re-evaluation           |

---

## Best Practices

1. **Confirm the org-level Shield Advanced subscription** before creating the policy.
2. **Enable auto-remediation + AWS Config** so enrollment actually happens.
3. **Pair with an FMS WAF policy** for L7 protection.
4. **Associate Route 53 health checks** on critical resources for health-based detection.
5. **Scope by OU/tags/type** deliberately.
6. **Configure SRT + proactive engagement** at the org level separately.
7. **Monitor compliance** and alert on unprotected critical resources.

---

## Useful Links

- [Shield Advanced policies in Firewall Manager](https://docs.aws.amazon.com/waf/latest/developerguide/getting-started-fms-shield.html)
- [Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
- [FMS policy scope](https://docs.aws.amazon.com/waf/latest/developerguide/working-with-policies.html)
- [FMS compliance](https://docs.aws.amazon.com/waf/latest/developerguide/fms-compliance.html)

---
