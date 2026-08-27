# AWS Network Firewall - Cheat Sheet

## Overview

**AWS Network Firewall (ANFW)** is a managed, stateful network firewall and IPS/IDS for VPC traffic. It inspects Layer 3–7 traffic using **Suricata-compatible** rules, supports domain filtering, and deploys as **firewall endpoints** in dedicated subnets that traffic is routed through.

**Key point:** Network Firewall is **inline via routing** — you must steer traffic to the firewall endpoint using VPC/TGW route tables. It doesn't see traffic unless routes send it there.

---

## Core Concepts

| Component               | Description                                                             |
| ----------------------- | ----------------------------------------------------------------------- |
| **Firewall**            | The resource, associated with a VPC; has endpoints per AZ               |
| **Firewall endpoint**   | An interface in a dedicated firewall subnet in each AZ                   |
| **Firewall policy**     | Container that references rule groups + default actions                 |
| **Rule group**          | Stateless or stateful set of rules                                      |
| **Stateless rules**     | Fast 5-tuple match; forward/drop/pass to stateful engine                |
| **Stateful rules**      | Suricata-compatible; connection-aware, domain, protocol detection       |
| **Rule order**          | Stateful evaluation: **default action order** or **strict order**       |

---

## Stateless vs Stateful

| Aspect            | Stateless rule group                    | Stateful rule group                          |
| ----------------- | --------------------------------------- | -------------------------------------------- |
| Matching          | 5-tuple (IP/port/protocol)              | Connection-aware, Suricata rules, domains    |
| Actions           | pass / drop / forward-to-stateful       | pass / drop / alert / reject                 |
| Capacity unit     | Rule capacity (reserved at creation)    | Rule capacity (reserved at creation)         |
| Use               | Fast pre-filter                         | Deep inspection, IDS/IPS, domain filtering   |

> Stateless rules run first and can **forward** flows to the stateful engine. Stateful rules do the deep work (Suricata signatures, domain allow/deny, TLS SNI).

---

## Stateful Rule Types

| Type                        | Description                                                 |
| --------------------------- | ----------------------------------------------------------- |
| **5-tuple**                 | Suricata-style IP/port/protocol rules                       |
| **Domain list**             | Allow/deny by domain (HTTP Host / TLS SNI)                  |
| **Suricata compatible**     | Raw Suricata rule strings (full signature power)            |
| **AWS managed rule groups** | Curated threat signatures (e.g., botnet, malware, Abused domains) |

### Rule Evaluation Order (stateful)

| Order mode        | Behavior                                                        |
| ----------------- | --------------------------------------------------------------- |
| **Default (action) order** | Rules grouped by action; pass → drop → alert priority applied by engine defaults |
| **Strict order**  | Rules evaluated in the exact order you set; explicit default actions (drop/alert established) |

> **Strict order** gives predictable, sequential evaluation (like a traditional firewall) and is generally recommended for clear policy.

---

## How It Works (routing)

```
Ingress from IGW / spoke VPC
        |
        v  (route table steers traffic to firewall endpoint)
Firewall endpoint (dedicated firewall subnet, per AZ)
        |
        ├── Stateless rules (5-tuple): pass / drop / forward-to-stateful
        |
        ├── Stateful rules (Suricata/domain): pass / drop / alert / reject
        |
        v
Onward to destination (workload subnet / NAT / TGW)
```

Typical **distributed** deployment (single VPC):
- **Firewall subnet** per AZ (small, dedicated).
- **IGW route table** (edge association): route workload CIDRs → firewall endpoint.
- **Workload subnet** route table: `0.0.0.0/0` → firewall endpoint.
- **Firewall subnet** route table: `0.0.0.0/0` → IGW (or NAT).

Typical **centralized** deployment: inspection VPC attached to TGW with **appliance mode** on the TGW attachment.

Newer models: a **Transit Gateway-attached firewall** (native TGW attachment — no inspection VPC or GWLB endpoints) and, for endpoint-based firewalls, **VPC endpoint associations** to add endpoints across VPCs/AZs/accounts. See the **Deployment Models**, **Associations**, and **Symmetric Routing & Troubleshooting** sheets.

**Deployment mode (set at creation, immutable):** the default **source-preservation** mode keeps the original client source IP and is inline via routing. A newer **no-source-preservation** mode (public preview, **us-east-2 only**) makes the firewall an **explicit forward proxy** — it attaches to a **NAT gateway**, terminates and re-establishes client connections (destinations see the NAT gateway IP), and clients reach it via a **proxy FQDN** using proxy env vars instead of route tables. See the **Deployment Models** sheet.

**Flow operations:** you can inspect and manage the firewall's **state table** (the table of stateful flows) using async **flow capture** and **flow flush** operations. See the **Symmetric Routing & Troubleshooting** sheet.

This service's cheat sheets: Overview · Stateless Rule Groups · Stateful Rules & Default Actions · Suricata · Firewall Policy · Deployment Models · Associations · TLS Inspection · Logging & Monitoring · Symmetric Routing & Troubleshooting.

---

## CLI

```bash
# Create a stateful rule group (Suricata strict order example)
aws network-firewall create-rule-group \
  --rule-group-name deny-bad-domains --type STATEFUL --capacity 100 \
  --rule-group '{"RulesSource":{"RulesSourceList":{"TargetTypes":["TLS_SNI","HTTP_HOST"],"Targets":["badsite.example.com"],"GeneratedRulesType":"DENYLIST"}}}'

# Create a firewall policy referencing the rule group
aws network-firewall create-firewall-policy \
  --firewall-policy-name my-policy \
  --firewall-policy '{"StatelessDefaultActions":["aws:forward_to_sfe"],"StatelessFragmentDefaultActions":["aws:forward_to_sfe"],"StatefulRuleGroupReferences":[{"ResourceArn":"arn:aws:network-firewall:...:stateful-rulegroup/deny-bad-domains"}]}'

# Create the firewall (endpoints in dedicated firewall subnets)
aws network-firewall create-firewall \
  --firewall-name my-fw --firewall-policy-arn arn:... \
  --vpc-id vpc-0abc \
  --subnet-mappings SubnetId=subnet-fw-1a SubnetId=subnet-fw-1b

# Enable logging (ALERT and/or FLOW to S3/CW/Firehose)
aws network-firewall update-logging-configuration \
  --firewall-arn arn:... \
  --logging-configuration '{"LogDestinationConfigs":[{"LogType":"ALERT","LogDestinationType":"S3","LogDestination":{"bucketName":"aws-nfw-logs"}}]}'
```

---

## Logging

| Log type | Contents                                            |
| -------- | --------------------------------------------------- |
| **ALERT**| Traffic matching rules with `alert`/`drop` action   |
| **FLOW** | Connection metadata for traffic sent to stateful engine |
| **TLS**  | TLS inspection logs (when TLS inspection configured) |

Destinations: CloudWatch Logs, S3, Firehose.

---

## Pricing

| Item                     | Cost                                          |
| ------------------------ | --------------------------------------------- |
| Firewall endpoint        | **Hourly charge per endpoint (per AZ)**       |
| Data processed           | **Per-GB inspected**                          |
| Logging                  | Destination (CW/S3/Firehose) charges          |

> You pay **per endpoint-hour per AZ + per-GB processed**. A multi-AZ deployment multiplies the hourly cost by the number of AZs.

---

## Quotas (defaults, adjustable)

| Resource                                | Default limit |
| --------------------------------------- | ------------- |
| Firewalls per account per Region        | (adjustable)  |
| Rule groups per account                 | (adjustable)  |
| Rule group capacity (reserved at create)| fixed at creation — plan ahead |
| Firewall policies per account           | (adjustable)  |

---

## Gotchas & Caveats

1. **It only inspects traffic you route to it.** Network Firewall is inline via route tables — if routes don't send traffic through the firewall endpoint, it's simply bypassed.
2. **Rule group capacity is reserved at creation and can't be increased in place** — you must recreate the rule group with higher capacity. Estimate generously.
3. **Default vs strict rule order changes behavior** — default (action) order can produce surprising precedence; strict order is predictable but requires explicit default actions.
4. **Asymmetric routing breaks stateful inspection** — return traffic must traverse the same firewall/AZ. In TGW designs, enable **appliance mode**; in single-VPC, use per-AZ endpoints and same-AZ routing.
5. **Domain filtering uses HTTP Host and TLS SNI** — it does **not** decrypt TLS by default; encrypted-SNI/ESNI or IP-only traffic can evade domain rules unless you enable TLS inspection.
6. **Needs dedicated firewall subnets** — don't put workloads in the firewall subnet; the endpoint consumes the subnet.
7. **Per-AZ endpoints are required for AZ resilience** — a single endpoint is an AZ SPOF, and traffic in an AZ without an endpoint can't be inspected/delivered.
8. **The stateless default action must forward to the stateful engine** (`aws:forward_to_sfe`) for stateful rules to see traffic.
9. **Managed rule groups update over time** — signatures change; test in alert mode before dropping.
10. **TLS inspection has cost and complexity** — certificate management, and some traffic can't be inspected.
11. **Route-table edge association** (gateway route table on the IGW) is what forces ingress through the firewall — forgetting it lets inbound bypass inspection.
12. **Suricata rule syntax errors** silently fail rule-group creation/update — validate rules.

---

## Troubleshooting

| Issue                                       | Cause                                           | Fix                                                        |
| ------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| Firewall "isn't scanning" traffic           | Routes don't send traffic to the endpoint        | Fix VPC/IGW/TGW route tables to steer through firewall     |
| Return traffic dropped                      | Asymmetric routing across AZs                    | Same-AZ endpoints; enable TGW appliance mode               |
| Domain block not working                    | Encrypted SNI / IP-based access / no TLS inspection | Add IP rules; consider TLS inspection                    |
| Rule group won't grow                       | Capacity fixed at creation                       | Recreate with higher capacity                              |
| Stateful rules never match                  | Stateless default not forwarding to SFE          | Set stateless default to `aws:forward_to_sfe`             |
| Inbound bypasses firewall                   | Missing IGW edge route association               | Associate a gateway route table on the IGW                 |
| Unexpected allow/deny precedence            | Default (action) order semantics                 | Switch to strict order with explicit defaults              |

---

## Best Practices

1. **Deploy one firewall endpoint per AZ** and route same-AZ to avoid asymmetry and AZ SPOFs.
2. **Use strict rule order** for predictable, auditable policy.
3. **Start rules in `alert` mode**, review ALERT logs, then switch to `drop`.
4. **Reserve generous rule-group capacity** at creation.
5. **Enable ALERT + FLOW logging** to S3 (+ Athena) for visibility and tuning.
6. **Use AWS managed rule groups** for baseline threat coverage; layer domain lists for policy.
7. **Enable appliance mode** on TGW attachments for centralized inspection.
8. **Keep firewall subnets dedicated and small**; never place workloads there.
9. **Consider TLS inspection** where you need real domain/content enforcement, weighing cost/complexity.

---

## Useful Links

- [What is AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html)
- [How it works](https://docs.aws.amazon.com/network-firewall/latest/developerguide/how-it-works.html)
- [Rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-groups.html)
- [Stateful rule evaluation order](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-rule-evaluation-order.html)
- [Routing / deployment models](https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html)
- [Appliance mode with TGW](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html)
- [Logging](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-logging.html)
- [Network Firewall quotas](https://docs.aws.amazon.com/network-firewall/latest/developerguide/quotas.html)

---
