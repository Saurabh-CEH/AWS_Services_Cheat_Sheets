# AWS Transit Gateway - Cheat Sheet

## Overview

**AWS Transit Gateway (TGW)** is a regional network hub that connects VPCs, VPNs, Direct Connect gateways, and peered TGWs in a hub-and-spoke topology. It replaces complex meshes of VPC peerings with centralized, **transitive** routing controlled by TGW route tables.

**Key point:** TGW is **Region-scoped** and routing is governed by **TGW route tables** via **associations** (which route table an attachment uses) and **propagations** (which routes an attachment advertises).

---

## Core Concepts

| Component                | Description                                                              |
| ------------------------ | ------------------------------------------------------------------------ |
| **Transit Gateway**      | The regional routing hub                                                 |
| **Attachment**           | A connection to the TGW (VPC, VPN, Direct Connect GW, TGW peering, Connect) |
| **TGW route table**      | Controls routing between attachments                                     |
| **Association**          | Binds an attachment to exactly **one** TGW route table (its routing domain) |
| **Propagation**          | Auto-advertises an attachment's routes (CIDRs) into a TGW route table    |
| **TGW peering**          | Connects two TGWs (same or cross-Region/account)                         |
| **Appliance mode**       | Keeps a flow on the same TGW ENI/AZ for stateful appliance inspection    |
| **Connect attachment**   | GRE/BGP overlay for SD-WAN appliances                                    |

---

## How Routing Works

```
Attachment (VPC/VPN/DXGW) ── associated with ──► ONE TGW route table
        |
        v
TGW route table has routes (static + propagated) → target attachments
        |
        ├── match → forward to target attachment
        └── no match / blackhole → dropped
```

Two independent knobs per attachment:
- **Association** = which route table this attachment *uses* to make forwarding decisions (one only).
- **Propagation** = which route tables *learn* this attachment's CIDRs (can be many).

> Segmentation (e.g., prod vs dev, or a shared-services domain) is built by creating multiple TGW route tables and controlling associations/propagations.

---

## Common Topologies

### Flat (any-to-any)
All VPCs associate + propagate to a single TGW route table → full transitive connectivity.

### Segmented (isolated environments)
```
Prod RT:   prod VPCs associate here; propagate prod CIDRs
Dev RT:    dev VPCs associate here;  propagate dev CIDRs
Shared RT: shared-services VPC; both prod & dev propagate to Shared, Shared propagates to both
Result:    prod↔shared and dev↔shared, but prod ✗ dev
```

### Centralized egress / inspection
Spoke VPCs default-route `0.0.0.0/0 → TGW`; a central egress/inspection VPC (NAT or Network Firewall) sends traffic out and back.

---

## CLI

```bash
# Create a TGW (optionally disable default RT association/propagation for control)
aws ec2 create-transit-gateway \
  --options DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable,DnsSupport=enable

# Attach a VPC (specify one subnet per AZ you want to route through)
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0abc --vpc-id vpc-0abc \
  --subnet-ids subnet-1a subnet-1b

# Create a route table and associate/propagate
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-0abc
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id tgw-rtb-prod --transit-gateway-attachment-id tgw-attach-0abc
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id tgw-rtb-shared --transit-gateway-attachment-id tgw-attach-0abc

# Static route (e.g., default route to an egress VPC attachment)
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-spokes \
  --destination-cidr-block 0.0.0.0/0 \
  --transit-gateway-attachment-id tgw-attach-egress

# Don't forget the VPC side: subnet route table → TGW
aws ec2 create-route --route-table-id rtb-vpc \
  --destination-cidr-block 10.0.0.0/8 --transit-gateway-id tgw-0abc
```

---

## Pricing

| Item                          | Cost                                          |
| ----------------------------- | --------------------------------------------- |
| Attachment                    | **Hourly charge per attachment**              |
| Data processed                | **Per-GB processed by the TGW**               |
| TGW peering data              | Per-GB + inter-Region transfer for cross-Region |

> You pay **per attachment-hour AND per-GB processed** — traffic between two VPCs on the same TGW is billed on both the ingress processing and any cross-AZ/Region transfer. Consolidate and be deliberate about routing everything through TGW.

---

## Quotas (defaults, adjustable)

| Resource                                     | Default limit |
| -------------------------------------------- | ------------- |
| Attachments per TGW                          | 5,000         |
| TGW route tables per TGW                      | 20            |
| Routes per TGW route table                    | 10,000        |
| VPCs that can attach                          | thousands (via attachments) |
| Bandwidth per VPC attachment                  | up to ~100 Gbps (burst) |
| Bandwidth per VPN tunnel                      | ~1.25 Gbps per tunnel |

---

## Gotchas & Caveats

1. **An attachment associates with exactly ONE route table** — segmentation is designed via associations/propagations, not multiple associations.
2. **You must add routes on BOTH the VPC side and the TGW side** — a TGW route table entry alone doesn't move traffic; the VPC subnet route table also needs a route to the TGW.
3. **TGW is non-transitive across peering by default** — routes do **not** propagate over TGW **peering**; you must add **static routes** on each TGW for peered destinations.
4. **VPC attachment uses one subnet per AZ** — TGW places an ENI in that subnet; traffic to an AZ without an attachment subnet can't be delivered there.
5. **Appliance mode is required for stateful inspection across AZs** — without it, TGW may hash flows to different AZs and break stateful firewalls (asymmetric routing).
6. **Same-CIDR/overlapping VPCs still can't route** — TGW doesn't do NAT; overlapping CIDRs need other solutions (PrivateLink, NAT).
7. **Default route table association/propagation is ON unless you disable it at creation** — leaving it on can accidentally create any-to-any connectivity. Disable for segmented designs.
8. **Cross-account attachments need RAM sharing** — share the TGW via Resource Access Manager, then the other account creates the attachment.
9. **Security Groups don't span TGW like peering** — SG referencing across a TGW is limited/unsupported vs same-VPC; use CIDRs.
10. **Blackhole routes** are how you explicitly drop traffic — useful, but easy to forget and cause silent drops.
11. **Data-processing charges apply to all TGW traffic** — routing intra-Region VPC-to-VPC through TGW costs more than direct peering for simple cases.
12. **DNS/appliance-mode/multicast are per-TGW options** — must be enabled at the right level; multicast has its own constraints.

---

## Troubleshooting

| Issue                                        | Cause                                          | Fix                                                        |
| -------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| VPCs attached but can't reach each other     | Missing route on VPC side or TGW route table    | Add route VPC→TGW and ensure TGW RT has the destination    |
| Traffic to peered TGW fails                  | Peering routes aren't propagated                | Add static routes for peered CIDRs on each TGW             |
| Stateful firewall drops return traffic       | Asymmetric AZ routing                            | Enable **appliance mode** on the inspection attachment     |
| Prod can unexpectedly reach dev              | Default association/propagation left enabled     | Disable defaults; use separate RTs with scoped propagation |
| One AZ can't reach across TGW                | No attachment subnet in that AZ                  | Add an attachment subnet in the missing AZ                 |
| Cross-account attach fails                   | TGW not shared via RAM                            | Share TGW with the account via RAM, then attach            |
| Silent drops                                 | Blackhole route or no matching route             | Check TGW route table for blackhole/missing entries        |

---

## Best Practices

1. **Disable default association/propagation** and design route tables explicitly for segmentation.
2. **Use separate route tables** per environment (prod/dev/shared) for isolation.
3. **Add routes on both VPC and TGW sides** as a standard checklist.
4. **Enable appliance mode** for any centralized stateful inspection VPC.
5. **Add static routes for TGW peering** — propagation doesn't cross peerings.
6. **One attachment subnet per AZ** you serve, in a small dedicated subnet.
7. **Centralize egress/inspection** through a shared VPC to reduce NAT/firewall sprawl.
8. **Share via RAM** for multi-account; keep TGW ownership in a networking account.
9. **Watch data-processing cost** — don't route trivial intra-Region flows through TGW when peering suffices.

---

## Useful Links

- [What is a Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)
- [How TGW works](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
- [TGW route tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Associations & propagations](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html#tgw-route-tables-association-propagation)
- [Appliance mode](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html)
- [TGW peering](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)
- [TGW quotas](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html)

---
