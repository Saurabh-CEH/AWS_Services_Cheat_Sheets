# AWS Firewall Manager - Security Group Policies Cheat Sheet

## Overview

**FMS security group policies** centrally manage security groups across an Organization. There are three modes: **Common** (deploy a baseline SG to resources), **Content audit** (enforce allowed/denied SG rules), and **Usage audit** (find and clean up unused/redundant SGs).

**Key point:** SG policies are the main way to enforce network guardrails org-wide — e.g., "no `0.0.0.0/0` on port 22" or "attach this baseline SG everywhere" — with FMS auto-remediating drift.

---

## Policy Modes

| Mode                | Purpose                                                              |
| ------------------- | -------------------------------------------------------------------- |
| **Common**          | Apply a **primary/baseline** SG (replica) to in-scope resources      |
| **Content audit**   | Audit/enforce SG **rules** — allowed rules only, or denied rules blocked |
| **Usage audit**     | Detect **unused/redundant** SGs and (optionally) remediate them      |

---

## Common Security Group Policy

- You define a **primary security group** in the FMS admin account.
- FMS creates a **replica** of it in each in-scope account/VPC and associates it with in-scope resources (e.g., ENIs, instances).
- Ensures a consistent baseline SG everywhere.

## Content Audit Policy

- Define **allowed** rules (only these are permitted) or **denied** rules (these must not exist).
- FMS flags non-compliant SGs; with remediation on, it can remove violating rules.
- Great for guardrails like "deny inbound `0.0.0.0/0` to 22/3389."

## Usage Audit Policy

- Finds SGs that are **unused** (not associated) or **redundant** (duplicate rule sets).
- Helps clean up SG sprawl (relevant to the SG-per-ENI and rules-per-SG limits).

---

## How It Works

```
FMS admin defines an SG policy (mode + scope + remediation)
        |
        ├── Common       → replicate primary SG → associate to in-scope resources
        ├── Content audit→ compare SG rules to allow/deny lists → flag/remove violations
        └── Usage audit  → find unused/redundant SGs → flag/clean up
        |
        v
Compliance dashboard + (optional) auto-remediation on existing + future resources
```

---

## CLI

```bash
aws fms list-policies
aws fms get-policy --policy-id <sg-policy-id>
aws fms list-compliance-status --policy-id <sg-policy-id>

# Put an SG policy — SecurityServicePolicyData.Type =
#   "SECURITY_GROUPS_COMMON" | "SECURITY_GROUPS_CONTENT_AUDIT" | "SECURITY_GROUPS_USAGE_AUDIT"
aws fms put-policy --policy file://sg-policy.json
```

---

## Gotchas & Caveats

1. **Content-audit remediation can DELETE rules** — enforcing "denied rules" with remediation on removes violating rules automatically; test in audit-only first.
2. **Common policy replicates a primary SG** — changing the primary propagates everywhere; a mistake has broad blast radius.
3. **Auto-remediation must be ON to enforce** — otherwise it only reports non-compliance.
4. **AWS Config required** in scoped accounts/Regions.
5. **SG limits interact** — usage-audit cleanup helps with SG-per-ENI (5→16) and rules-per-SG (60) limits; enforcement can bump into them.
6. **Managed (replicated) SGs shouldn't be hand-edited** — FMS reverts drift on the replicas.
7. **Scope precision matters** — a broad content-audit deny can break legitimate access org-wide; scope and test.
8. **Order of rollout** — enable Config + trusted access before creating policies.
9. **Usage audit "redundant" detection** is heuristic — review before auto-deleting.
10. **Replica SGs consume SG quota** in member accounts — factor into SG-per-VPC limits.
11. **Regional** — SG policies are per-Region; deploy per Region as needed.
12. **De-scoping** may leave replicated SGs behind — verify cleanup.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Legit access broken after content audit     | Deny rule too broad + remediation on            | Narrow the audit; test in audit-only mode first           |
| Baseline SG not applied                     | Remediation off / Config off / scope            | Enable remediation + Config; fix scope                    |
| Replicated SG keeps reverting my edits      | It's FMS-managed                                | Change the primary SG in the FMS policy                   |
| Cleanup deleted a needed SG                 | Usage-audit false "unused/redundant"            | Review before enabling auto-remediation                   |
| SG quota errors                             | Replicas + rules hit limits                     | Consolidate; request increases; run usage audit           |
| Not applied in a Region                     | Policy is per-Region                            | Create the policy in each Region                          |

---

## Best Practices

1. **Run content/usage audits in audit-only mode first**, review findings, then enable remediation.
2. **Use content-audit deny policies** for high-value guardrails (block `0.0.0.0/0` to 22/3389).
3. **Treat replicated/common SGs as read-only**; edit the primary.
4. **Use usage audit** to control SG sprawl and stay within limits.
5. **Scope narrowly** and expand after validation.
6. **Enable Config + trusted access** before policies.
7. **Deploy per Region**; monitor compliance dashboards.

---

## Useful Links

- [Security group policies](https://docs.aws.amazon.com/waf/latest/developerguide/security-group-policies.html)
- [Common security group policy](https://docs.aws.amazon.com/waf/latest/developerguide/security-group-policies.html#security-group-policies-common)
- [Content audit policy](https://docs.aws.amazon.com/waf/latest/developerguide/security-group-policies.html#security-group-policies-audit)
- [Usage audit policy](https://docs.aws.amazon.com/waf/latest/developerguide/security-group-policies.html#security-group-policies-usage)

---
