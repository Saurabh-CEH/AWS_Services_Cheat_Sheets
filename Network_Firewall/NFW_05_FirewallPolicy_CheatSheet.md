# AWS Network Firewall - Firewall Policy Cheat Sheet

## Overview

A **firewall policy** ties together stateless + stateful rule group references and default actions; a **firewall** applies a policy and creates **endpoints**. This sheet focuses on the **policy** itself. Deployment models and associations have their own sheets.

**Key point:** The policy's **stateless default must forward to the stateful engine** (`aws:forward_to_sfe`) for stateful rules to run, and the **rule order + stateful default actions** are set here (see the Stateful Rules & Default Actions sheet for the full default-action guide).

**Related sheets:** Deployment Models · Associations · Stateful Rules & Default Actions · Stateless Rule Groups · Suricata · Symmetric Routing & Troubleshooting.

---

## Firewall Policy Contents

| Element                          | Description                                                  |
| -------------------------------- | ------------------------------------------------------------ |
| **Stateless rule group refs**    | Fast 5-tuple pre-filter groups                               |
| **Stateless default actions**    | Action for unmatched packets (usually `aws:forward_to_sfe`)  |
| **Stateless fragment defaults**  | Action for fragmented packets                                |
| **Stateful rule group refs**     | Deep-inspection groups (Suricata, domain, managed)           |
| **Stateful default actions**     | (Strict order) e.g., drop/alert established                  |
| **Rule order**                   | Default (action) order or strict order                       |
| **Policy variables**             | `$HOME_NET` etc.                                             |

---

## Deployment & Associations (see dedicated sheets)

Network Firewall supports **distributed**, **centralized (inspection VPC + TGW)**, **Transit Gateway-attached (native)**, and **decentralized ingress** models. Full comparison, diagrams, and a when-to-use decision guide are in the dedicated **Deployment Models** sheet (`NFW_DeploymentModels_CheatSheet.md`).

For how firewalls link to policies, subnets/endpoints, VPC endpoint associations, TGW attachments, RAM shares, and **container associations** (ECS/EKS dynamic IP sets), see the dedicated **Associations** sheet (`NFW_Associations_CheatSheet.md`).

> Whatever the model: Network Firewall only inspects traffic you **route to its endpoint**, and routing must be **symmetric** (see the Symmetric Routing & Troubleshooting sheet).

---

## CLI

```bash
# Create the firewall policy
aws network-firewall create-firewall-policy \
  --firewall-policy-name my-policy \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulRuleGroupReferences": [{"ResourceArn":"arn:...:stateful-rulegroup/deny-domains"}],
    "StatefulEngineOptions": {"RuleOrder":"STRICT_ORDER"},
    "StatefulDefaultActions": ["aws:drop_established","aws:alert_established"]
  }'

# Create the firewall with endpoints in dedicated firewall subnets (per AZ)
aws network-firewall create-firewall \
  --firewall-name my-fw --firewall-policy-arn arn:... \
  --vpc-id vpc-0abc \
  --subnet-mappings SubnetId=subnet-fw-1a SubnetId=subnet-fw-1b \
  --delete-protection --subnet-change-protection

# Get the endpoint IDs to use as route targets
aws network-firewall describe-firewall --firewall-name my-fw
```

---

## Pricing

| Item                     | Cost                                          |
| ------------------------ | --------------------------------------------- |
| Firewall endpoint        | Hourly per endpoint (per AZ)                  |
| Data processed           | Per-GB inspected                              |

> Multi-AZ deployment multiplies the hourly endpoint cost by the number of AZs. Centralized inspection consolidates endpoints but concentrates data-processing charges.

---

## Gotchas & Caveats

1. **Only inspects routed traffic** — the firewall is bypassed entirely if route tables don't send traffic to its endpoint.
2. **IGW edge route table association is required for ingress inspection** — without it, inbound internet traffic skips the firewall.
3. **Dedicated firewall subnets** — never place workloads there; the endpoint owns the subnet.
4. **Per-AZ endpoints for resilience** — a single endpoint is an AZ SPOF; traffic in an AZ without an endpoint can't be inspected/delivered.
5. **Centralized model needs appliance mode** on the inspection VPC's TGW attachment, or asymmetric routing breaks stateful inspection.
6. **Delete protection & subnet-change protection** are opt-in — enable them to avoid accidental teardown/traffic disruption.
7. **Stateless default must forward to SFE** for stateful rules to run.
8. **Strict order requires stateful default actions** — set drop/alert established explicitly.
9. **Endpoint creation takes time** and consumes subnet IPs — size firewall subnets accordingly.
10. **Changing subnet mappings is disruptive** — adding/removing AZs affects traffic.
11. **Return path design** — for centralized egress/inspection, ensure return traffic comes back through the same firewall.
12. **Policy changes propagate to all endpoints** — validate in alert mode before enforcing drops fleet-wide.
13. **TGW-attached firewalls don't support VPC endpoint associations** — if you need multi-VPC/multi-account endpoints, use the endpoint-based (GWLBe) model; if you want no inspection VPC, use the TGW-attached model. You can't mix the two on one firewall.
14. **TGW-attached firewall still needs correct TGW routing** — associate the firewall's TGW attachment in the right route tables so forward and return traffic both traverse it; appliance-mode-style symmetry is handled by the native attachment but routing must send traffic there.
15. **Constrained Availability Zones aren't supported** — you can't place a firewall endpoint in a constrained AZ ("Availability Zone is not supported by Network Firewall"); use a supported AZ and keep resources in the same zone.
16. **Decentralized ingress needs the IGW edge association; NAT-downstream needs the NAT subnet routed through the firewall** — otherwise ingress or egress bypasses inspection.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Firewall isn't inspecting traffic           | Routes don't steer traffic to the endpoint      | Fix VPC/IGW/TGW routes to send traffic through firewall    |
| Inbound bypasses firewall                   | Missing IGW edge route table association         | Associate a gateway route table on the IGW                 |
| Return traffic dropped                      | Asymmetric routing (centralized)                | Enable appliance mode; same-AZ routing (distributed)       |
| One AZ not inspected                        | No firewall endpoint in that AZ                  | Add an endpoint (subnet mapping) in the AZ                 |
| Stateful rules never run                    | Stateless default not `aws:forward_to_sfe`      | Set stateless default to forward                           |
| Accidental firewall/subnet deletion         | Protections disabled                             | Enable delete + subnet-change protection                  |

---

## Best Practices

1. **One firewall endpoint per AZ**; route same-AZ to avoid asymmetry and SPOFs.
2. **Associate an IGW edge route table** so ingress is inspected.
3. **Stateless default = `aws:forward_to_sfe`**; strict order with explicit stateful defaults.
4. **Enable appliance mode** for centralized inspection via TGW.
5. **Use dedicated, adequately-sized firewall subnets** (workload-free).
6. **Turn on delete/subnet-change protection.**
7. **Roll out policy changes in alert mode** before enforcing.
8. **Design and test return paths** for egress/inspection.

---

## Useful Links

- [Firewall policies](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-policies.html)
- [Firewalls](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewalls.html)
- [Deployment models / architectures](https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html)
- [Route table configuration](https://docs.aws.amazon.com/network-firewall/latest/developerguide/route-table-configuration.html)
- [Appliance mode with TGW](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html)
- [Deployment with a gateway route table](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-standard-architectures.html)

---
