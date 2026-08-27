# AWS Network Firewall - Associations Cheat Sheet

## Overview

"Association" in Network Firewall covers the several ways resources are linked together: the **firewall ↔ policy**, the **firewall ↔ subnet/endpoint**, **VPC endpoint associations** (extra endpoints across VPCs/AZs/accounts), the **Transit Gateway attachment** (for TGW-attached firewalls), **RAM resource shares** for cross-account use, and **container associations** (dynamic IP sets tracking running ECS/EKS container IPs).

**Key point:** **Container associations do exist** — they don't inspect container traffic directly; they build a **dynamic IP set** of running container IPs (from ECS/EKS lifecycle events) that your **stateful rules** reference. Your firewall endpoints still inspect traffic via routing as usual.

---

## Association Types at a Glance

| Association                    | Links                                              | Notes                                             |
| ------------------------------ | -------------------------------------------------- | ------------------------------------------------- |
| **Firewall ↔ Firewall policy** | A firewall uses exactly one policy at a time       | Change/swap the policy to change behavior         |
| **Firewall ↔ Subnet (endpoint)** | Subnet mappings create firewall endpoints (per AZ) | Dedicated firewall subnets; one endpoint per subnet |
| **VPC endpoint association**   | Adds a firewall endpoint in another subnet/VPC/account | Endpoint-based firewalls only (NOT TGW-attached)  |
| **TGW attachment**             | A TGW-attached firewall attaches to a Transit Gateway | Native model; no inspection VPC/GWLBe             |
| **RAM resource share**         | Shares firewall / firewall policy / rule groups cross-account | Enables multi-account use                        |
| **Container association**      | Tracks running **ECS/EKS** container IPs as a **dynamic IP set** referenced by stateful rules | Not a traffic path — an IP-set feed              |
| **Rule group ↔ policy (reference)** | Policy references stateless/stateful rule groups | Not an "association" API, but a linkage           |

> Container associations feed a **dynamic IP set**; you still route container **subnet/VPC/TGW** traffic through a firewall endpoint for inspection (see Deployment Models).

---

## Firewall ↔ Policy

- A firewall is associated with **one firewall policy**; the policy holds the rule groups + default actions.
- You can **swap** the associated policy (e.g., dev → prod policy) — changes propagate to all the firewall's endpoints.

```bash
aws network-firewall associate-firewall-policy \
  --firewall-name my-fw \
  --firewall-policy-arn arn:aws:network-firewall:...:firewall-policy/prod-policy
```

---

## Firewall ↔ Subnet (Firewall Endpoints)

- **Subnet mappings** define where firewall **endpoints** are created — **one per AZ**, in **dedicated firewall subnets**.
- Adding/removing subnet mappings adds/removes endpoints (disruptive to traffic in those AZs).

```bash
# Endpoints via subnet mappings at creation
aws network-firewall create-firewall \
  --firewall-name my-fw --firewall-policy-arn arn:... --vpc-id vpc-0abc \
  --subnet-mappings SubnetId=subnet-fw-1a SubnetId=subnet-fw-1b

# Add an AZ later
aws network-firewall associate-subnets \
  --firewall-name my-fw --subnet-mappings SubnetId=subnet-fw-1c
```

---

## VPC Endpoint Associations

Extend an **endpoint-based** firewall with endpoints in **other subnets, VPCs, AZs, or accounts** the firewall is shared with.

| Capability                                | Detail                                                     |
| ----------------------------------------- | ---------------------------------------------------------- |
| Endpoints in **other VPCs**               | Beyond the primary protected VPC                           |
| **Multiple endpoints in one AZ**          | Supported                                                  |
| **Cross-account** endpoints               | In accounts the firewall is shared with (RAM)              |
| **Availability**                          | Endpoint-based firewalls **only** — not TGW-attached       |

```bash
aws network-firewall create-vpc-endpoint-association \
  --firewall-arn arn:aws:network-firewall:...:firewall/my-fw \
  --vpc-id vpc-other \
  --subnet-mapping SubnetId=subnet-fw-other
```

> **Not available for transit gateway-attached firewalls.** Choose the endpoint-based model if you need this.

---

## Transit Gateway Attachment (TGW-attached firewalls)

- A **TGW-attached firewall** associates directly with a **Transit Gateway** (owned or RAM-shared) — no inspection VPC or GWLB endpoints.
- You then associate/propagate the firewall's **TGW attachment** in TGW route tables like any other attachment (see the TGW Routing sheet).

```bash
# Create a TGW-attached firewall against a TGW you own or that's shared with you
aws network-firewall create-firewall \
  --firewall-name tgw-fw --firewall-policy-arn arn:... \
  --transit-gateway-id tgw-0abc
```

---

## RAM Sharing (cross-account)

- Share the **firewall**, **firewall policy**, and/or **rule groups** via **AWS Resource Access Manager** so other accounts can use them (and, for endpoint-based firewalls, create VPC endpoint associations in their subnets).
- For TGW-attached firewalls, the **TGW** itself is shared via RAM so the firewall can attach.

---

## Container Associations (ECS / EKS)

A **container association** collects the IP addresses of **running containers** in your **Amazon ECS** and **Amazon EKS** clusters and maintains them as a **dynamic IP set**. You reference that IP set in stateful rules, so you can match traffic by container source/destination IP **without hardcoding addresses** as pods/tasks come and go.

| Aspect                        | Detail                                                              |
| ----------------------------- | ------------------------------------------------------------------- |
| **What it does**              | Subscribes to container **lifecycle events** (ECS task start/stop, EKS pod start/stop) and builds/maintains an IP set |
| **What it does NOT do**       | No interaction with cluster traffic/networking — it only collects IPs; your endpoints still inspect traffic via routing |
| **Container types**           | `ECS` or `EKS` — **set at creation, cannot be changed**             |
| **Monitoring configurations** | Up to **5** per association; each targets one cluster ARN (+ optional attribute filters) |
| **Service-linked role**       | `AWSServiceRoleForNetworkFirewall` auto-created on first use (needs `iam:CreateServiceLinkedRole`) |
| **How ECS is tracked**        | Network Firewall creates a **service-managed EventBridge Managed Rule** (`NetworkFirewallManagedRule-<cluster>-<hash>`) — don't modify it |
| **How EKS is tracked**        | Subscribes to pod lifecycle via the EKS Pulse Event Service         |

### Referencing in a rule group (ReferenceSets)

Add the container-association ARN to the rule group's `ReferenceSets.IPSetReferences` to create a named Suricata variable (e.g., `@CONTAINER_IPS`):

```bash
aws network-firewall create-rule-group \
  --rule-group-name my-container-rules --type STATEFUL --capacity 100 \
  --rule-group '{
    "RulesSource": { "RulesString": "alert tcp @CONTAINER_IPS any -> any any (sid:1; rev:1;)" },
    "ReferenceSets": {
      "IPSetReferences": {
        "CONTAINER_IPS": { "ReferenceArn": "arn:aws:network-firewall:us-east-1:123456789012:container-association/my-ecs-monitor" }
      }
    }
  }'
```

### Create / update / delete

```bash
# Create (ECS example with an attribute filter — EC2 launch type only)
aws network-firewall create-container-association \
  --container-association-name my-ecs-monitor --type ECS \
  --container-monitoring-configurations '[{"ClusterArn":"arn:aws:ecs:us-east-1:123456789012:cluster/my-cluster","AttributeFilters":[{"Key":"ecs.instance-type","Value":"c5.xlarge"}]}]'

# Update (type is fixed; needs the update token from describe — optimistic concurrency)
aws network-firewall update-container-association \
  --container-association-arn arn:...:container-association/my-ecs-monitor \
  --type ECS --update-token "TOKEN" --container-monitoring-configurations '[...]'

# Delete (async; transitions to DELETING; must remove rule-group references first)
aws network-firewall delete-container-association \
  --container-association-arn arn:...:container-association/my-ecs-monitor
```

### Networking requirements & limits

| Requirement / limit                        | Detail                                                          |
| ------------------------------------------ | --------------------------------------------------------------- |
| **EKS: disable SNAT** on the VPC CNI       | So the firewall sees the **pod's original IP**, not the node IP — otherwise rules referencing the association don't match |
| **ECS: `awsvpc` network mode only**        | `bridge` and `host` network modes are **not supported**         |
| **ECS Fargate + attribute filters**        | Attribute filters match **EC2 launch type only**; Fargate tasks have no container-instance attributes — **attach no attributes** to track Fargate IPs |
| **IPSet reference exclusivity**            | A rule group's IPSet references are **either all regular IP sets/prefix lists OR all container associations** — mixing returns `InvalidRequestException` |
| **Clusters must be same Region + account** | As the container association                                    |

### Quotas

| Resource                                          | Default | Adjustable |
| ------------------------------------------------- | ------- | ---------- |
| Container associations per account per Region     | 100     | Yes        |
| Monitoring configurations per container association | 5     | No         |
| Container association references per rule group   | 30      | No         |

> The **5 IP-set-references-per-rule-group** limit does **not** apply to container-association references (they have their own limit of 30).

### Status values

`CREATING` → `ACTIVE` (tracking) · `UPDATING` (old config keeps running) · `DELETING` (async cleanup; can't update in this state).

---

## Gotchas & Caveats

1. **Container associations feed an IP set, not a traffic path** — they track ECS/EKS container IPs for rules to reference; you still route container traffic through a firewall endpoint for inspection.
2. **EKS: SNAT must be disabled** on the VPC CNI, or the firewall sees the node IP (not the pod IP) and container-association rules won't match.
3. **ECS: only `awsvpc` network mode** is supported for container associations (not `bridge`/`host`); Fargate attribute filters don't apply (attach no attributes to track Fargate IPs).
4. **A rule group can't mix regular IP sets and container associations** — references are all one type or all the other, else `InvalidRequestException`.
5. **Container type is fixed at creation** (ECS vs EKS) and can't be changed; you must recreate to switch.
6. **Can't delete a container association still referenced** by a rule group (`InvalidOperationException`) — remove references first; deletion is async (DELETING).
7. **Don't touch the service-managed EventBridge Managed Rule** (`NetworkFirewallManagedRule-...`) that ECS associations create — NFW manages its lifecycle.
8. **VPC endpoint associations are endpoint-based-firewall only** — TGW-attached firewalls don't support them.
9. **One policy per firewall** — you can't associate two policies; consolidate rule groups into one policy.
10. **One firewall endpoint per subnet, one subnet per AZ** for the base firewall — use VPC endpoint associations to add more endpoints/AZs/VPCs.
11. **Dedicated firewall subnets** — the endpoint owns the subnet; don't run workloads there.
12. **Adding/removing subnet mappings is disruptive** to traffic in those AZs.
13. **Cross-account associations require RAM sharing first** — share the firewall/policy/rule group (or TGW) before the other account can use it.
14. **Deleting a firewall/policy that's still associated fails** — disassociate/remove references first.
15. **Constrained AZs** can't host firewall endpoints — pick supported AZs.
16. **Swapping the associated policy** applies fleet-wide across the firewall's endpoints — validate in alert mode first.

---

## Troubleshooting

| Issue                                          | Cause                                           | Fix                                                        |
| ---------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| Can't add endpoints in another VPC/account     | Trying it on a TGW-attached firewall            | Use endpoint-based firewall + VPC endpoint association     |
| Can't attach firewall to a TGW                 | TGW not owned / not shared via RAM              | Share the TGW via RAM, then create the TGW-attached firewall |
| Endpoint not created in an AZ                  | No subnet mapping in that AZ / constrained AZ    | Add a subnet mapping in a supported AZ                     |
| Can't delete firewall/policy                   | Still associated/referenced                      | Disassociate resources; remove policy/rule-group references |
| Cross-account use fails                        | No RAM share                                     | Share firewall/policy/rule group (or TGW) via RAM          |
| Orphaned rule group association                | Reference left after policy/Region change        | Locate and remove the dangling reference                   |
| Container-association rules don't match (EKS)  | SNAT enabled → firewall sees node IP, not pod IP  | Disable SNAT on the VPC CNI                                 |
| Container-association rules don't match (ECS)  | Non-`awsvpc` mode, or Fargate + attribute filters | Use `awsvpc` mode; drop attribute filters for Fargate      |
| `InvalidRequestException` on rule group        | Mixing regular IP sets + container associations   | Keep references all one type per rule group                |
| Can't delete container association             | Still referenced by a rule group                  | Remove rule-group references first (delete is async)       |
| `AccessDeniedException` creating association    | Missing `ecs:DescribeClusters`/`eks:DescribeCluster`/`iam:CreateServiceLinkedRole` | Grant the required permissions |

---

## Best Practices

1. **Use container associations** to match ECS/EKS traffic by dynamic container IPs — and still route that traffic through a firewall endpoint for inspection.
2. **For EKS, disable SNAT** and **for ECS, use `awsvpc` mode** so the firewall sees real container IPs.
3. **Use VPC endpoint associations** for multi-VPC/account inspection on endpoint-based firewalls.
3. **Use the TGW-attached model** when you want centralized inspection without an inspection VPC (and don't need VPC endpoint associations).
4. **One endpoint per AZ** you serve; add more via subnet mappings / VPC endpoint associations.
5. **Share via RAM** from a central networking/security account for multi-account.
6. **Keep firewall subnets dedicated**; validate AZ support.
7. **Audit for orphaned associations/references** during cleanups.
8. **Validate policy swaps in alert mode** before enforcing across all endpoints.

---

## Useful Links

- [Firewalls and firewall endpoints](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewalls.html)
- [Creating a VPC endpoint association](https://docs.aws.amazon.com/network-firewall/latest/developerguide/creating-vpc-endpoint-association.html)
- [Creating a firewall (incl. TGW-attached)](https://docs.aws.amazon.com/network-firewall/latest/developerguide/creating-firewall.html)
- [Associate/disassociate subnets](https://docs.aws.amazon.com/network-firewall/latest/APIReference/API_AssociateSubnets.html)
- [Sharing rule groups / firewalls with RAM](https://docs.aws.amazon.com/network-firewall/latest/developerguide/sharing.html)
- [Container associations (ECS/EKS)](https://docs.aws.amazon.com/network-firewall/latest/developerguide/container-associations.html)
- [TGW-attached firewall (blog)](https://aws.amazon.com/blogs/security/why-and-how-to-migrate-to-a-transit-gateway-attached-aws-network-firewall/)

---
