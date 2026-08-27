# AWS Network Firewall - Deployment Models Cheat Sheet

## Overview

Network Firewall is **inline via routing** — you must route traffic to a firewall endpoint for it to be inspected. The **deployment model** you choose determines where endpoints live, how routing is done, cost, and blast radius. This sheet compares the models and helps you pick.

**Key point:** Regardless of model, the golden rule is **symmetric routing** — forward and return traffic of a flow must reach the **same** firewall endpoint. See the Symmetric Routing & Troubleshooting sheet.

---

## Model Comparison

| Model                              | Where endpoints live                     | Routing you manage                          | Best for                                  |
| ---------------------------------- | ---------------------------------------- | ------------------------------------------- | ----------------------------------------- |
| **Distributed (per-VPC)**          | Firewall subnet(s) in each protected VPC | VPC subnet RTs + IGW edge RT per VPC        | Few VPCs; strong per-VPC isolation        |
| **Centralized (inspection VPC + TGW, GWLBe)** | Firewall subnet(s) in a central inspection VPC | TGW route tables + inspection VPC RTs; appliance mode | Many VPCs; consolidated inspection        |
| **Transit Gateway-attached (native)** | Managed by AWS (no inspection VPC)    | TGW route tables (associate the NFW attachment) | Many VPCs; simplest centralized, no GWLBe plumbing |
| **Decentralized ingress**          | Firewall subnet per AZ in the VPC        | IGW **edge/gateway** route table + app subnet RT | Internet ingress inspection at the edge   |

---

## Distributed (per-VPC)

```
Firewall subnet (per AZ)  ── firewall endpoint
IGW edge route table:  workload CIDRs → firewall endpoint
Workload subnet RT:    0.0.0.0/0 → firewall endpoint
Firewall subnet RT:    0.0.0.0/0 → IGW (or NAT)
```
- Each VPC inspects its own traffic; nothing shared.
- Simple mental model, but **endpoints (and cost) replicate per VPC**.

## Centralized (inspection VPC + TGW, GWLBe-routed)

```
Spoke VPCs → TGW → inspection VPC (firewall endpoints) → TGW → destination
Inspection VPC's TGW attachment: appliance mode ENABLED
```
- One inspection VPC serves many spokes (east-west and/or egress).
- **Requires TGW appliance mode** so flows stay on one AZ/endpoint (symmetry).
- You manage TGW route tables + inspection VPC route tables to steer traffic through the endpoints.

## Transit Gateway-attached firewall (native)

```
Spoke VPCs → TGW ── (native Network Firewall attachment) ── AWS Network Firewall
```
- The firewall **attaches directly to a Transit Gateway** you own (or one shared via RAM) — **no inspection VPC, no GWLB endpoints, no GWLBe routing**.
- AWS manages the underlying inspection/symmetry plumbing; you associate the firewall's TGW attachment in TGW route tables like any other attachment.
- Simplest way to do centralized inspection at scale.
- **Caveat:** **VPC endpoint associations are NOT available** for TGW-attached firewalls (those are for endpoint-based firewalls).

## Decentralized ingress (internet-facing)

```
Internet → IGW → (IGW edge route table) → firewall endpoint → ALB/workload
```
- Use an **IGW edge (gateway) route table association** to force inbound internet traffic through the firewall endpoint, plus an outbound route in the app subnet.
- If the firewall is **downstream of a NAT gateway**, ensure the **NAT gateway's subnet** routes through the firewall endpoint.

---

## Deployment Mode: Source-Preservation vs No-Source-Preservation

Beyond *where* you place endpoints, Network Firewall has two **deployment modes** chosen **at firewall creation** (immutable — you cannot change a firewall's mode afterward):

| Aspect                | Source-preservation (default)                        | No-source-preservation (forward proxy)                        |
| --------------------- | ---------------------------------------------------- | ------------------------------------------------------------- |
| Original source IP    | **Preserved** through inspection                     | **Not preserved** — firewall re-originates with NAT gateway IP |
| Traffic path          | VPC route tables steer traffic to firewall endpoint  | Clients send traffic to the firewall (proxy) — **no route changes** |
| Connection handling   | Transparent (inline), single connection              | Firewall **terminates** the client connection and **re-establishes** a new one to the destination |
| Attaches to           | Firewall subnet endpoints                            | A **NAT gateway** (firewall uses the NAT GW's IP for egress)  |
| How clients reach it  | Routing                                              | Proxy **FQDN (DnsName)** + proxy env vars (`http_proxy`/`https_proxy`) |
| Stateless engine      | Runs (5-tuple pre-filter)                            | **Not used** for explicit-proxy traffic — CONNECT goes straight to stateful engine |
| Multi-VPC reach       | VPC endpoint associations                            | VPC endpoint associations (auto private hosted zones for FQDN) |
| Availability          | GA (all Regions)                                     | **Public preview, us-east-2 (Ohio) only** — subject to change |

### No-source-preservation (explicit forward proxy)

```
App (http_proxy → firewall FQDN:3128 / https_proxy → :8443)
   │  CONNECT request
   ▼
No-source-preservation firewall (stateful inspection, terminates connection)
   │  re-establishes new connection using NAT gateway IP
   ▼
NAT gateway → Internet destination
```

- The firewall acts as an **explicit proxy**: applications set proxy environment variables to the firewall's **DnsName (FQDN)**; **no route table changes** are needed.
- Default proxy listener ports: **HTTP `3128`**, **HTTPS `8443`** (configurable).
- **Endpoint must be in the same AZ as the attached NAT gateway**, and in a **different subnet** from the NAT gateway subnet.
- Extend to other VPCs with **`CreateVPCEndpointAssociation`** — Network Firewall auto-creates **private hosted zones** so the FQDN resolves in each associated VPC.
- After creation, firewall transitions **PROVISIONING → READY** and the **DnsName** appears; if it goes **FAILED**, delete and recreate.
- **Choose it when** you need an explicit proxy that terminates/re-originates connections, want client-side proxy config instead of routing, or need the source IP hidden behind the NAT gateway. Do **not** pick it if you must preserve original client source IPs.

---

## Routing Rules of Thumb

| Traffic direction        | Route to make it inspected                                    |
| ------------------------ | ------------------------------------------------------------- |
| Ingress from internet    | IGW **edge/gateway route table** → workload CIDRs → firewall endpoint |
| Egress from workloads    | Workload subnet RT → `0.0.0.0/0` → firewall endpoint          |
| Firewall → out           | Firewall subnet RT → `0.0.0.0/0` → IGW/NAT                     |
| East-west (spokes)       | TGW route tables send inter-spoke traffic via inspection/NFW attachment |
| NAT downstream           | NAT gateway's subnet RT → firewall endpoint                   |

---

## Choosing a Model (decision guide)

```
One or few VPCs, want isolation?           → Distributed
Many VPCs, want central inspection,
   OK managing inspection VPC + GWLBe?     → Centralized (inspection VPC + TGW)
Many VPCs, want central inspection,
   want least plumbing (no inspection VPC)?→ Transit Gateway-attached (native)
Need endpoints in multiple VPCs/accounts
   for one firewall?                       → Endpoint-based firewall + VPC endpoint associations
                                             (NOT TGW-attached)
Internet ingress inspection at the edge?   → Decentralized ingress (IGW edge route table)
```

| Factor            | Distributed | Centralized (GWLBe) | TGW-attached (native) |
| ----------------- | ----------- | ------------------- | --------------------- |
| Inspection VPC    | No          | Yes                 | No                    |
| GWLBe routing     | Per VPC     | Yes (manual)        | No (AWS-managed)      |
| Appliance mode    | N/A         | Required            | Handled by attachment |
| Endpoint cost     | Per VPC × AZ| Central × AZ        | Managed               |
| VPC endpoint associations | N/A | Supported           | **Not supported**     |
| Scales to many VPCs | Poorly    | Well                | Well                  |

---

## Gotchas & Caveats

1. **Only inspects routed traffic** — wrong/missing routes = firewall bypassed entirely (check `ReceivedPackets`).
2. **Symmetric routing is mandatory** — centralized needs **appliance mode**; distributed needs per-AZ endpoints + same-AZ routing.
3. **IGW edge route table association is required for ingress inspection** — without it, inbound skips the firewall.
4. **TGW-attached firewalls don't support VPC endpoint associations** — pick the endpoint-based model if you need multi-VPC/account endpoints.
5. **Dedicated firewall subnets** — never place workloads there; the endpoint consumes the subnet.
6. **Per-AZ endpoints for resilience** — an AZ without an endpoint can't be inspected/delivered.
7. **Constrained AZs aren't supported** — place endpoints only in supported AZs.
8. **Centralized concentrates cost + blast radius** — one policy change affects all spokes; validate in alert mode.
9. **NAT-downstream requires the NAT subnet routed through the firewall**, or egress bypasses inspection.
10. **Switching models later is disruptive** — plan the model up front.
11. **Deployment mode is immutable** — source-preservation vs no-source-preservation is fixed at firewall creation; to switch you must create a new firewall.
12. **No-source-preservation is preview / us-east-2 only** and does **not** preserve client source IPs (destinations see the NAT gateway IP); its stateless engine is bypassed for explicit-proxy traffic.

---

## Best Practices

1. **Pick the model deliberately** using the decision guide; don't retrofit.
2. **Enable appliance mode** for centralized inspection VPCs.
3. **Use the TGW-attached model** to avoid inspection-VPC/GWLBe complexity when you don't need VPC endpoint associations.
4. **One endpoint per AZ** you serve; route same-AZ.
5. **Associate an IGW edge route table** for ingress inspection.
6. **Keep firewall subnets dedicated** and adequately sized.
7. **Validate routing symmetry** (see the Symmetric Routing sheet) before enforcing drops.

---

## Useful Links

- [Network Firewall deployment models / architectures](https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html)
- [Standard architectures (gateway route table)](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-standard-architectures.html)
- [Appliance mode with TGW](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html)
- [Why migrate to a Transit Gateway-attached firewall (blog)](https://aws.amazon.com/blogs/security/why-and-how-to-migrate-to-a-transit-gateway-attached-aws-network-firewall/)
- [Avoiding asymmetric routing](https://docs.aws.amazon.com/network-firewall/latest/developerguide/asymmetric-routing.html)
- [Centralized inspection (whitepaper)](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/network-firewall.html)
- [No-source-preservation mode (forward proxy)](https://docs.aws.amazon.com/network-firewall/latest/developerguide/nfw-no-source-preservation.html)
- [Getting started with no-source-preservation mode](https://docs.aws.amazon.com/network-firewall/latest/developerguide/nfw-nosource-getting-started.html)

---
