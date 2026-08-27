# AWS Firewall Manager - Compliance & Remediation Cheat Sheet

## Overview

FMS continuously evaluates in-scope accounts/resources against each policy and reports **compliance status**. With **auto-remediation** enabled, FMS actively brings non-compliant resources into line (attaching Web ACLs, creating protections/firewalls, fixing SG rules). This sheet covers how evaluation, remediation, and drift work — and the operational gotchas.

**Key point:** FMS relies on **AWS Config** to evaluate resources, and **auto-remediation** is what turns a policy from "report-only" into active enforcement. Both must be right for FMS to work.

---

## Evaluation & Remediation Flow

```
AWS Config records resource state in each account/Region
        |
        v
FMS evaluates in-scope resources against the policy
        |
        ├── compliant     → no action
        └── non-compliant → remediation OFF: report only
                            remediation ON:  FMS fixes (attach/create/modify) — existing + future
        |
        v
Compliance dashboard per policy / account / resource
```

---

## Compliance Concepts

| Concept                  | Detail                                                          |
| ------------------------ | --------------------------------------------------------------- |
| **Compliance status**    | Per account and per resource, for each policy                   |
| **Remediation (auto)**   | On = FMS enforces; Off = FMS reports only                       |
| **Remediation eligibility** | Some findings are auto-remediable, some require manual action |
| **Drift**                | Manual changes to FMS-managed resources are reverted on next evaluation |
| **Scope**                | Accounts (OU/IDs), resource types, tags (include/exclude)       |

---

## CLI

```bash
# Compliance overview for a policy
aws fms list-compliance-status --policy-id <id>

# Detailed compliance for a specific member account
aws fms get-compliance-detail --policy-id <id> --member-account 222233334444

# Admin / org status
aws fms get-admin-account
aws fms list-member-accounts

# Notifications: FMS can publish compliance to SNS (configure in policy/console)
```

---

## Gotchas & Caveats

1. **AWS Config must be enabled** in every account/Region FMS manages — no Config, no evaluation or remediation there. This is the most common "policy not working" cause.
2. **Auto-remediation OFF = report-only** — FMS won't attach Web ACLs, create protections, or fix SGs; it only flags. Turn it on to enforce.
3. **Drift is reverted** — hand-edits to FMS-managed resources (Web ACLs, replicated SGs, firewalls) are undone on the next evaluation. Change the policy, not the resource.
4. **De-scoping/removal behavior varies** — removing a resource/account from scope may leave managed resources (Web ACLs, SGs, firewalls) behind, or remove protections; the "WebACLs not removed after de-scoping" issue is common. Verify cleanup.
5. **`FMS ThrottlingException`** — large orgs hit API throttling during evaluation/remediation; use backoff and expect eventual consistency.
6. **Evaluation isn't instant** — there's lag between a change and FMS reflecting it; "non-compliant but looks fine" often means evaluation hasn't caught up.
7. **Cross-account logging/bucket statements** — FMS WAF policies can add S3 bucket-policy statements for logging, causing IaC drift; reconcile.
8. **Trusted access + delegated admin required** — set up Organizations trusted access and the FMS admin account before policies work.
9. **Not all findings auto-remediate** — some require manual intervention; the dashboard indicates eligibility.
10. **Per-Region policies** — regional resources need policies per Region; compliance is reported per Region.
11. **Member accounts can't disable FMS enforcement** — by design; escalate policy changes to the FMS admin.
12. **Notifications need setup** — wire compliance to SNS/EventBridge to get alerted rather than polling the dashboard.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Policy shows nothing / not enforcing        | Config disabled / remediation off / scope       | Enable Config; turn on remediation; fix scope             |
| Resource keeps getting reconfigured         | FMS reverts drift on managed resources          | Change the policy, not the resource                       |
| Managed resource left after de-scoping      | Cleanup behavior                                 | Manually remove; verify policy settings                   |
| `ThrottlingException`                       | Org-scale API throttling                         | Backoff/retry; stagger; request limit increase            |
| "Non-compliant" but resource looks correct  | Evaluation lag / Config not recording            | Verify Config recording; wait for re-evaluation           |
| Terraform drift                             | FMS added logging/bucket statements              | Reconcile IaC with FMS-managed changes                    |
| No alerts on non-compliance                 | Notifications not configured                     | Publish compliance to SNS/EventBridge                     |

---

## Best Practices

1. **Enable AWS Config everywhere** FMS will manage, before creating policies.
2. **Roll out with remediation OFF first** (report-only), validate scope, then enable enforcement.
3. **Treat FMS-managed resources as read-only**; change behavior via the policy.
4. **Wire compliance to SNS/EventBridge** for proactive alerting.
5. **Handle throttling** with backoff; expect eventual consistency at scale.
6. **Reconcile IaC** with FMS-managed logging/bucket statements to avoid drift.
7. **Verify cleanup** when de-scoping resources/accounts.
8. **Document that FMS is authoritative** so teams don't fight remediation.

---

## Useful Links

- [FMS compliance](https://docs.aws.amazon.com/waf/latest/developerguide/fms-compliance.html)
- [Remediation](https://docs.aws.amazon.com/waf/latest/developerguide/fms-remediation.html)
- [FMS prerequisites (Config, trusted access)](https://docs.aws.amazon.com/waf/latest/developerguide/fms-prereq.html)
- [Working with FMS policies](https://docs.aws.amazon.com/waf/latest/developerguide/working-with-policies.html)
- [FMS notifications](https://docs.aws.amazon.com/waf/latest/developerguide/fms-notifications.html)

---
