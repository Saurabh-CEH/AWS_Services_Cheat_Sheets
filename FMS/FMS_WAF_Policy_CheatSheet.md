# AWS Firewall Manager - WAF Policy Cheat Sheet

## Overview

An **FMS WAF policy** centrally deploys and enforces AWS WAF (WAFv2) Web ACLs and rule groups across in-scope accounts and resources (ALB, CloudFront, API Gateway, etc.) from the FMS admin account. FMS applies the policy to **existing and future** resources and (with auto-remediation) keeps them compliant.

**Key point:** An FMS WAF policy creates and manages Web ACLs in member accounts. Those Web ACLs are **FMS-managed** — hand-edits get reverted, and de-scoping/removal behavior is a common operational surprise.

---

## Policy Structure

| Element                         | Description                                                    |
| ------------------------------- | -------------------------------------------------------------- |
| **Pre-process rule groups**     | Rule groups evaluated **before** any account-added rules       |
| **Post-process rule groups**    | Rule groups evaluated **after** account-added rules            |
| **Account-managed rules space** | Optional room for member accounts to add their own rules       |
| **Default action**              | Allow/Block for the managed Web ACL                            |
| **Resource type & scope**       | Which resource types/accounts/tags the policy applies to       |
| **Logging config**              | Centralized WAF logging (destination, redaction)               |

---

## How It Works

```
FMS admin defines WAF policy (rule groups + scope + remediation)
        |
        v
FMS creates/attaches a MANAGED Web ACL on in-scope resources (existing + future)
        |
        ├── pre-process rule groups run first
        ├── (optional) account's own rules
        └── post-process rule groups run last
        |
        v
Auto-remediation keeps resources associated & compliant; drift is reverted
```

---

## CLI

```bash
# List / inspect FMS policies (in the FMS admin account)
aws fms list-policies
aws fms get-policy --policy-id <id>

# Compliance for the policy
aws fms list-compliance-status --policy-id <id>
aws fms get-compliance-detail --policy-id <id> --member-account 222233334444

# Put a WAF policy (JSON is large — usually via console/IaC).
# SecurityServicePolicyData.Type = "WAFV2"; ManagedServiceData carries the rule groups.
aws fms put-policy --policy file://waf-policy.json
```

---

## Gotchas & Caveats

1. **FMS creates and OWNS the Web ACL in member accounts** — teams often see a Web ACL they didn't create; hand-edits are reverted on the next evaluation.
2. **"Custom Web ACL missing in accounts"** usually means the resource is out of policy scope, or a different FMS policy manages it — check scope and tags.
3. **Pre-process vs post-process placement matters** — pre-process rules run before account rules, post-process after; misplacing rule groups changes precedence.
4. **Auto-remediation must be ON to enforce** — with it off, FMS only reports non-compliance and won't attach/fix Web ACLs.
5. **AWS Config must be enabled** in each account/Region, or FMS can't evaluate/remediate there.
6. **FMS-managed logging can add S3 bucket-policy statements** — a frequent cause of Terraform/IaC drift; reconcile your IaC.
7. **Removing a resource from scope may leave or remove the Web ACL** depending on config — the "WebACLs not removed after de-scoping" complaint is common; verify cleanup behavior.
8. **Regional vs global** — CloudFront is global (us-east-1); regional resources need per-Region policies.
9. **WCU budget applies** — the managed Web ACL + your rules must fit the WCU cap.
10. **Throttling at scale** — large orgs can hit `ThrottlingException` during evaluation.
11. **Members can't fully edit the managed Web ACL** — only within the account-managed rules space (if allowed).
12. **Policy changes propagate on FMS's schedule** — not instant; allow time for enforcement across accounts.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| WAF policy not applying                     | Config off / scope wrong / no remediation       | Enable Config; fix scope; enable auto-remediation          |
| Custom Web ACL "missing"/reverted           | FMS scope/remediation manages it                | Review policy scope; exclude by tag if intentional         |
| Terraform drift after policy                | FMS added logging/bucket-policy statements      | Reconcile IaC to include FMS-managed statements            |
| Web ACL not removed after de-scoping        | Cleanup/config behavior                          | Manually remove leftover Web ACLs; verify settings         |
| Rules run in wrong order                    | Pre/post-process placement                       | Move rule groups to correct pre/post slot                  |
| `ThrottlingException`                       | Org-scale API throttling                         | Retries/backoff; stagger; request limit increase           |

---

## Best Practices

1. **Scope by OU and tags**, start narrow, expand after validating.
2. **Treat the FMS Web ACL as read-only** in member accounts; change via the policy.
3. **Place threat/baseline rule groups pre-process**, app-specific allowances post-process.
4. **Enable auto-remediation** once you trust the scope.
5. **Account for FMS-managed logging statements** in your IaC.
6. **Deploy per-Region policies** for regional resources; one for CloudFront.
7. **Watch WCU budget** for the managed Web ACL plus account rules.
8. **Monitor compliance dashboards** and alert on drift.

---

## Useful Links

- [AWS WAF policies in Firewall Manager](https://docs.aws.amazon.com/waf/latest/developerguide/waf-policies.html)
- [How AWS WAF policies work](https://docs.aws.amazon.com/waf/latest/developerguide/getting-started-fms-create-security-policy.html)
- [FMS compliance](https://docs.aws.amazon.com/waf/latest/developerguide/fms-compliance.html)
- [FMS prerequisites](https://docs.aws.amazon.com/waf/latest/developerguide/fms-prereq.html)

---
