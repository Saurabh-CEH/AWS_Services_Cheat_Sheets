# AWS Firewall Manager - Network Firewall & DNS Firewall Policies Cheat Sheet

## Overview

Firewall Manager can centrally deploy **AWS Network Firewall** and **Route 53 Resolver DNS Firewall** across an Organization. FMS provisions the firewalls/associations (existing and future VPCs) and enforces the configuration, so you don't build them per account.

**Key point:** FMS Network Firewall policies support **distributed**, **centralized**, and **import** models; DNS Firewall policies deploy Resolver DNS Firewall rule groups to VPCs org-wide. Both rely on the same FMS prerequisites (Organizations, delegated admin, Config).

---

## Network Firewall Policy

| Deployment model     | What FMS does                                                     |
| -------------------- | ----------------------------------------------------------------- |
| **Distributed**      | Deploys firewall endpoints in each in-scope VPC (per AZ)          |
| **Centralized**      | Deploys/inspects via a central inspection VPC (with TGW)          |
| **Import existing**  | Brings existing firewalls under FMS management                    |

- The policy references **stateless/stateful rule groups** and manages firewall creation, endpoint placement, and routing (for supported models).
- FMS can manage the **route configuration** to steer traffic through the firewall (model-dependent).

## DNS Firewall Policy

- Deploys **Route 53 Resolver DNS Firewall rule groups** and associates them to in-scope VPCs.
- Controls **rule group priority/association** org-wide.
- Enforces consistent DNS filtering (block malicious domains) across accounts.

---

## How It Works

```
FMS admin defines Network Firewall / DNS Firewall policy (rule groups + scope + model)
        |
        ├── Network Firewall → create firewall endpoints (distributed) or use central VPC
        │                       + manage routing to steer traffic through inspection
        └── DNS Firewall     → associate Resolver DNS Firewall rule groups to VPCs
        |
        v
Applies to existing + future VPCs; auto-remediation enforces; compliance reported
```

---

## CLI

```bash
aws fms list-policies
aws fms get-policy --policy-id <id>
aws fms list-compliance-status --policy-id <id>

# Put policies — SecurityServicePolicyData.Type =
#   "NETWORK_FIREWALL" | "DNS_FIREWALL"
aws fms put-policy --policy file://nfw-policy.json
aws fms put-policy --policy file://dnsfw-policy.json
```

---

## Gotchas & Caveats

1. **FMS manages Firewall Manager rule groups vs standalone** — FMS-deployed Network Firewall / DNS Firewall configs are centrally owned; edits in member accounts get reverted.
2. **"FMS not showing policies in use" / "firewall rules not scanning"** often trace to scope, remediation off, or routing not steering traffic — verify the policy scope and that routes send traffic through the firewall.
3. **Network Firewall only inspects routed traffic** — even via FMS, you must ensure routing (which FMS manages in some models) actually steers traffic to endpoints.
4. **Distributed model creates endpoints per VPC/AZ** — cost multiplies; centralized consolidates but needs TGW + appliance mode.
5. **Dedicated firewall subnets required** for endpoints — FMS needs subnet availability in scoped VPCs.
6. **DNS Firewall rule group priority/association** conflicts — deploying via FMS on top of existing associations can clash; plan priorities.
7. **Rule group capacity** (Network Firewall) is reserved at creation — plan capacity before FMS-wide rollout.
8. **AWS Config + trusted access required**; enable before creating policies.
9. **Import model caveats** — importing existing firewalls has constraints; not all configs import cleanly.
10. **Regional** — deploy per Region; global scope isn't automatic for regional firewall constructs.
11. **De-scoping cleanup** — removing a VPC from scope may leave endpoints/associations; verify.
12. **Throttling at org scale** during evaluation.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| FMS policy "not in use" / not applied       | Scope/remediation/Config                        | Fix scope; enable remediation + Config                    |
| Firewall not scanning traffic               | Routing not steering to endpoints               | Verify route config (managed model) / VPC routes          |
| DNS Firewall rules not taking effect        | Association/priority conflict                    | Resolve priorities; confirm VPC associations              |
| Endpoints not created                       | No dedicated firewall subnets in scoped VPCs     | Ensure subnet availability                                |
| Central model return-traffic issues         | No appliance mode / asymmetric routing           | Enable appliance mode on inspection TGW attachment        |
| Config imports incompletely                 | Import-model constraints                         | Review import limitations; reconcile manually             |

---

## Best Practices

1. **Enable Config + trusted access + delegated admin** before creating policies.
2. **Choose the deployment model deliberately** — centralized to control cost, distributed for isolation.
3. **Ensure routing steers traffic** through the firewall (managed or manual).
4. **Plan Network Firewall rule-group capacity** before org-wide rollout.
5. **Coordinate DNS Firewall priorities** to avoid association conflicts.
6. **Treat FMS-managed firewalls as read-only** in member accounts.
7. **Deploy per Region**; monitor compliance and routing health.
8. **Validate in a pilot OU** before broad enforcement.

---

## Useful Links

- [Network Firewall policies in FMS](https://docs.aws.amazon.com/waf/latest/developerguide/network-firewall-policies.html)
- [DNS Firewall policies in FMS](https://docs.aws.amazon.com/waf/latest/developerguide/dns-firewall-policies.html)
- [Network Firewall deployment models](https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html)
- [Route 53 Resolver DNS Firewall](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall.html)
- [FMS prerequisites](https://docs.aws.amazon.com/waf/latest/developerguide/fms-prereq.html)

---
