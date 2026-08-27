# AWS Firewall Manager - Cheat Sheet

## Overview

**AWS Firewall Manager (FMS)** centrally configures and enforces firewall/security policies across an **AWS Organization**. From a single delegated administrator account, you deploy and continuously enforce WAF, Shield Advanced, Network Firewall, security-group, and DNS Firewall policies across existing and future accounts and resources.

**Key point:** FMS requires **AWS Organizations + a delegated FMS admin account**, and continuously **auto-remediates** drift — resources that fall out of compliance are brought back in line automatically (per policy config).

---

## Prerequisites

| Requirement                          | Detail                                                       |
| ------------------------------------ | ------------------------------------------------------------ |
| **AWS Organizations**                | Must be enabled with all features                            |
| **FMS administrator account**        | Delegated by the Organizations management account            |
| **AWS Config**                       | Must be enabled in accounts/Regions FMS manages              |
| **Trusted access**                   | FMS enabled as a trusted service in the Organization         |

---

## Policy Types

| Policy type                     | Manages                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| **AWS WAF**                     | Web ACLs + rule groups on ALB/CloudFront/API GW/etc.           |
| **Shield Advanced**             | Shield Advanced protections across accounts                    |
| **Security group**              | Common/audit/usage security-group policies                     |
| **Network Firewall**            | ANFW deployments across VPCs (distributed/centralized)         |
| **DNS Firewall**                | Route 53 Resolver DNS Firewall rule groups                     |
| **Network ACL (VPC)**           | Baseline first/last inbound & outbound NACL rules across org subnets |
| **Third-party firewall**        | Palo Alto **Cloud NGFW** / **Fortigate CNF** via AWS Marketplace (vendor onboarding + subscription) |

---

## Network ACL Policy (baseline NACL enforcement)

An FMS **network ACL policy** centrally enforces a baseline set of **first** and **last** inbound/outbound NACL rules across in-scope org subnets, while letting member accounts add **custom rules in between**.

| Aspect                | Detail                                                                       |
| --------------------- | ---------------------------------------------------------------------------- |
| What you define       | Ordered **first** and **last** rules (inbound + outbound); scope; force-remediation on conflict |
| Custom-rule window    | Accounts add their own rules **numbered 5,000–32,000**, between the policy's first/last rules |
| Managed marker        | FMS-managed NACLs carry the **`FMManaged=true`** tag — **don't modify** it or the subnet↔NACL associations |
| Remediation           | Start with **auto-remediation disabled**, review effects, then enable; force-remediation resolves rule conflicts |
| Caveats               | **Slower** to apply than other policy types (EC2 NACL API rate limits); if a subnet is newly in scope of multiple NACL policies, the **oldest** policy wins; remediation temporarily **increases rule count** — leave headroom under NACL limits |

> To stop managing a subnet, **exclude it via policy scope** (e.g., tag + exclude tag) — never hand-edit the managed association.

---

## Third-Party Firewall Policies

FMS can centrally deploy **third-party firewalls** as a managed policy:

- **Palo Alto Networks Cloud NGFW** and **Fortigate Cloud Native Firewall (CNF)**.
- Requires **vendor onboarding** (the third-party firewall admin account) and an active **AWS Marketplace subscription** in member accounts.
- FMS deploys/associates the vendor firewall endpoints across in-scope VPCs like it does Network Firewall.
- Use the FMS `*ThirdPartyFirewall*` APIs (e.g., list third-party firewall policies) to manage them.

---

## Policy Scope

| Scope control            | Detail                                                          |
| ------------------------ | --------------------------------------------------------------- |
| **Accounts**             | All accounts, or include/exclude by OU or account ID            |
| **Resource type**        | e.g., ALB, CloudFront, VPC                                       |
| **Resource tags**        | Include/exclude resources by tag                                |
| **Auto-remediation**     | On/off — whether FMS actively enforces and fixes non-compliance |

---

## How It Works

```
FMS admin account defines a policy (scope + rules)
        |
        v
FMS evaluates in-scope accounts/resources (via AWS Config)
        |
        ├── compliant     → no action
        └── non-compliant → (if auto-remediation on) create/attach/fix the resource
        |
        v
Applies to EXISTING and FUTURE resources continuously
```

- FMS reports **compliance status** per account/resource.
- With **auto-remediation on**, FMS will attach WAF Web ACLs, deploy firewall endpoints, adjust security groups, etc., to enforce the policy — including on newly created resources.

---

## Security Group Policy Modes

| Mode                | Purpose                                                        |
| ------------------- | -------------------------------------------------------------- |
| **Common**          | Apply a baseline set of SGs to in-scope resources              |
| **Content audit**   | Audit/enforce allowed or denied SG rules (guardrails)          |
| **Usage audit**     | Find and clean up unused/redundant security groups             |

---

## CLI

```bash
# (In Organizations mgmt account) delegate the FMS admin
aws fms associate-admin-account --admin-account 111122223333

# List / get policies (run in the FMS admin account)
aws fms list-policies
aws fms get-policy --policy-id <id>

# Get per-account compliance for a policy
aws fms list-compliance-status --policy-id <id>
aws fms get-compliance-detail --policy-id <id> --member-account 222233334444

# Put a policy (WAF example) — usually via console or IaC due to JSON size
aws fms put-policy --policy file://waf-policy.json
```

---

## Pricing

| Item                     | Cost                                              |
| ------------------------ | ------------------------------------------------- |
| FMS policy fee           | **Per policy per Region per month**               |
| Underlying services      | You also pay for WAF, Shield Advanced, Network Firewall, etc. that FMS deploys |

> FMS bills **per policy per Region** on top of the underlying service costs. A WAF policy across many Regions multiplies the FMS fee, plus the WAF Web ACL/rule/request charges it creates.

---

## Quotas (defaults, adjustable)

| Resource                                | Default limit |
| --------------------------------------- | ------------- |
| Policies per organization/account       | (adjustable)  |
| Accounts per organization               | Organizations limits apply |

---

## Gotchas & Caveats

1. **Requires Organizations + a delegated FMS admin account** — you cannot use FMS standalone in a single account without Organizations.
2. **AWS Config must be enabled** in every account/Region you want FMS to manage — missing Config = FMS can't evaluate/remediate there.
3. **Auto-remediation can overwrite manual changes** — resources FMS manages (e.g., a WAF Web ACL) shouldn't be hand-edited in member accounts; FMS reverts drift. This is the classic "my Web ACL disappeared / reverted" issue.
4. **Custom/member-account Web ACLs can appear "missing" or unmanaged** — if a resource isn't in policy scope or the policy created its own Web ACL, expectations mismatch. Check the policy scope and remediation setting.
5. **FMS WAF policies can auto-manage bucket/logging config** — e.g., adding S3 bucket policy statements for logging, which can cause Terraform/IaC drift.
6. **Policy scope by tag/OU is powerful but easy to misconfigure** — too-broad scope enforces everywhere; too-narrow leaves gaps.
7. **Removing a resource from scope may remove the protection** — e.g., WebACLs not cleaned up, or protections dropped, depending on config (a known operational pain point).
8. **FMS is Region-scoped per policy** (except global resources like CloudFront) — you need policies per Region for regional resources.
9. **ThrottlingException** can occur at scale — large orgs may hit API throttling during evaluation; retries/backoff needed.
10. **Third-party firewall policies** depend on Marketplace subscriptions being active in member accounts.
11. **Order of operations matters** — enable trusted access and Config before creating policies, or evaluations fail.
12. **Shield Advanced FMS policy still requires the Shield Advanced subscription** at the org level.

---

## Troubleshooting

| Issue                                       | Cause                                           | Fix                                                        |
| ------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| WAF policy not applying to accounts         | Config disabled / account not in scope / no remediation | Enable Config; fix scope; turn on auto-remediation   |
| Custom Web ACL missing in member accounts   | Policy scope/remediation reverted it            | Review FMS policy scope; exclude by tag if intentional     |
| Terraform drift after FMS WAF policy        | FMS added bucket policy/logging statements      | Reconcile IaC to include FMS-managed statements            |
| WebACLs not removed after de-scoping        | Cleanup behavior/config                          | Manually remove leftover Web ACLs; check policy settings   |
| `ThrottlingException`                       | API throttling at org scale                      | Add retries/backoff; stagger; request limit increase       |
| FMS shows non-compliant but resource looks fine | Evaluation lag / Config not reporting        | Verify Config recording; wait for re-evaluation            |

---

## Best Practices

1. **Set up Organizations, delegate the FMS admin, and enable Config** everywhere first.
2. **Scope policies deliberately** by OU and tags; start narrow, expand after validation.
3. **Treat FMS-managed resources as read-only** in member accounts — don't hand-edit; change the policy instead.
4. **Account for FMS-managed logging/bucket statements** in your IaC to avoid drift.
5. **Use audit-mode SG policies** to find/clean unused SGs before enforcing.
6. **Deploy per-Region policies** for regional resources; one for global (CloudFront).
7. **Monitor compliance dashboards** and set up notifications for non-compliance.
8. **Keep underlying subscriptions active** (Shield Advanced, Marketplace firewalls) for dependent policies.
9. **Plan for cost** — per-policy-per-Region fees plus the services FMS provisions.

---

## Useful Links

- [What is AWS Firewall Manager](https://docs.aws.amazon.com/waf/latest/developerguide/fms-chapter.html)
- [FMS prerequisites](https://docs.aws.amazon.com/waf/latest/developerguide/fms-prereq.html)
- [Working with FMS policies](https://docs.aws.amazon.com/waf/latest/developerguide/working-with-policies.html)
- [Security group policies](https://docs.aws.amazon.com/waf/latest/developerguide/security-group-policies.html)
- [Network Firewall policies](https://docs.aws.amazon.com/waf/latest/developerguide/network-firewall-policies.html)
- [Network ACL policies](https://docs.aws.amazon.com/waf/latest/developerguide/network-acl-policies.html)
- [Third-party firewall: Palo Alto Cloud NGFW](https://docs.aws.amazon.com/waf/latest/developerguide/getting-started-fms-palo-alto-networks-cloud-ngfw.html)
- [Third-party firewall: Fortigate CNF](https://docs.aws.amazon.com/waf/latest/developerguide/getting-started-fms-fortigate-cnf.html)
- [FMS compliance & remediation](https://docs.aws.amazon.com/waf/latest/developerguide/fms-compliance.html)
- [Firewall Manager pricing](https://aws.amazon.com/firewall-manager/pricing/)

---
