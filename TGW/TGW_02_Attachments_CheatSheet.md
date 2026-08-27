# AWS Transit Gateway - Attachments Cheat Sheet

## Overview

An **attachment** is how something connects to a Transit Gateway. Each attachment type (VPC, VPN, Direct Connect gateway, TGW peering, Connect) has its own behavior for routing, availability, and bandwidth. Attachments are the billing unit and the thing you associate/propagate in TGW route tables.

**Key point:** A **VPC attachment uses one subnet per AZ** — TGW places an ENI in each chosen subnet, and only AZs with an attachment subnet can send/receive traffic to that VPC via the TGW.

---

## Attachment Types

| Type                      | Connects                                    | Notes                                              |
| ------------------------- | ------------------------------------------- | -------------------------------------------------- |
| **VPC**                   | A VPC to the TGW                            | One subnet per AZ; ENI per AZ                       |
| **VPN**                   | Site-to-Site VPN to the TGW                 | 2 tunnels per connection; ~1.25 Gbps per tunnel    |
| **Direct Connect gateway**| DX to the TGW (via transit VIF)             | Hybrid connectivity                                |
| **TGW peering**           | Another TGW (same/cross Region/account)     | Routes NOT auto-propagated — static routes required |
| **Connect**               | SD-WAN/third-party appliance via GRE + BGP  | Overlay on an existing VPC or DX attachment        |

---

## VPC Attachment Details

| Setting                    | Detail                                                          |
| -------------------------- | --------------------------------------------------------------- |
| **Subnets**                | One per AZ you want reachable; use a small dedicated subnet     |
| **Appliance mode**         | Enable for stateful inspection to keep flows on one AZ/ENI       |
| **DNS support**            | Optional; enables cross-VPC DNS resolution behaviors            |
| **IPv6 support**           | Per-attachment option                                            |
| **Security group referencing** | Supported for TGW attachments in a Region (with limits)     |

### AZ behavior

```
VPC has subnets in AZ-a, AZ-b, AZ-c
Attachment subnets chosen: AZ-a, AZ-b  (NOT AZ-c)
Result: resources in AZ-c can still egress via TGW only if their subnet routes to TGW,
        but TGW can only deliver INTO the VPC through AZ-a / AZ-b ENIs.
```

> Add an attachment subnet in **every AZ** where you run workloads, or traffic to those AZs traverses cross-AZ (cost) or fails.

---

## CLI

```bash
# VPC attachment (one subnet per AZ)
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0abc --vpc-id vpc-0abc \
  --subnet-ids subnet-a subnet-b \
  --options ApplianceModeSupport=enable,DnsSupport=enable,Ipv6Support=disable

# Modify (e.g., add appliance mode later, or add/remove subnets)
aws ec2 modify-transit-gateway-vpc-attachment \
  --transit-gateway-attachment-id tgw-attach-0abc \
  --add-subnet-ids subnet-c \
  --options ApplianceModeSupport=enable

# Connect attachment (GRE overlay on a transport attachment)
aws ec2 create-transit-gateway-connect \
  --transport-transit-gateway-attachment-id tgw-attach-0abc \
  --options Protocol=gre

# List / inspect
aws ec2 describe-transit-gateway-attachments \
  --filters Name=transit-gateway-id,Values=tgw-0abc
aws ec2 describe-transit-gateway-vpc-attachments
```

---

## Bandwidth

| Attachment          | Approx bandwidth                                    |
| ------------------- | --------------------------------------------------- |
| VPC attachment      | Up to ~100 Gbps (bursts; aggregate)                 |
| VPN attachment      | ~1.25 Gbps **per tunnel** (use ECMP for more)       |
| Connect attachment  | Depends on transport; GRE + BGP with ECMP           |
| Per flow (single)   | Limited (~5 Gbps per flow historically) — a single TCP flow won't exceed the per-flow cap |

> A single flow can't use the full attachment bandwidth — the ~5 Gbps per-flow limit means large single transfers won't saturate a 100 Gbps attachment. Parallelize.

---

## Pricing

| Item                          | Cost                                          |
| ----------------------------- | --------------------------------------------- |
| Per attachment                | **Hourly charge per attachment**              |
| Data processed                | Per-GB through the TGW                         |

---

## Quotas (defaults, adjustable)

| Resource                                     | Default limit |
| -------------------------------------------- | ------------- |
| Attachments per TGW                          | 5,000         |
| VPC attachments per VPC to a single TGW      | 1             |
| Subnets per VPC attachment                   | one per AZ    |
| Connect peers per Connect attachment         | 4             |

---

## Gotchas & Caveats

1. **One subnet per AZ per VPC attachment** — TGW delivers into a VPC only through AZs that have an attachment subnet; missing AZs cause cross-AZ hops or failure.
2. **A VPC can have only ONE attachment to a given TGW** — you can't create two VPC attachments from the same VPC to the same TGW.
3. **Appliance mode must be enabled explicitly** and can be set at creation or via modify — without it, stateful inspection breaks on multi-AZ flows (asymmetric routing).
4. **Per-flow bandwidth cap (~5 Gbps)** — a single connection won't use the whole attachment; parallelize large transfers.
5. **VPN tunnels are ~1.25 Gbps each** — for more throughput use multiple tunnels/connections with ECMP.
6. **Connect attachments ride on a transport attachment** (VPC or DX) — you need the underlying attachment first.
7. **Attachment subnets should be small and dedicated** — don't run workloads in them; the TGW ENI lives there.
8. **Deleting an attachment leaves stale routes** in TGW/VPC route tables — clean them up to avoid blackholes.
9. **Cross-account attachments require RAM sharing first** — the owner shares the TGW; the other account then creates the attachment (and must accept if auto-accept is off).
10. **Modifying subnets is disruptive** — adding/removing attachment subnets can briefly affect traffic in those AZs.
11. **IPv6 support is per-attachment** — enabling it on the TGW isn't enough; set it on the attachment and add IPv6 routes.
12. **Appliance mode can't fix asymmetry across separate attachments** — it keeps a flow's forwarding consistent within one attachment; multi-attachment designs still need careful routing.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Traffic to one AZ fails via TGW             | No attachment subnet in that AZ                 | Add an attachment subnet in the missing AZ                 |
| Stateful firewall drops return traffic      | Appliance mode off / asymmetric routing         | Enable appliance mode on the inspection attachment         |
| Can't create second VPC attachment          | One attachment per VPC per TGW                  | Use route tables/segmentation instead of a 2nd attachment  |
| Low throughput on a big transfer            | Per-flow cap (~5 Gbps)                          | Parallelize connections; use multiple flows                |
| VPN throughput capped                       | ~1.25 Gbps per tunnel                           | Add tunnels/connections + ECMP                             |
| Cross-account attach fails                  | TGW not shared via RAM / not accepted           | Share via RAM; accept the attachment                       |
| Blackholed traffic after teardown           | Stale route pointing at deleted attachment      | Remove stale routes in TGW/VPC route tables                |

---

## Best Practices

1. **Add an attachment subnet in every AZ** you serve, using small dedicated subnets.
2. **Enable appliance mode** on any attachment fronting a stateful appliance/firewall.
3. **Parallelize large transfers** to work around the per-flow bandwidth cap.
4. **Use ECMP with multiple VPN tunnels/connections** for higher hybrid throughput.
5. **Share the TGW via RAM** from a central networking account for cross-account attachments.
6. **Keep attachment subnets workload-free.**
7. **Clean up stale routes** when deleting attachments.
8. **Plan Connect attachments** on top of a stable transport attachment.

---

## Useful Links

- [Transit gateway attachments](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)
- [VPC attachments](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)
- [Connect attachments](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-connect.html)
- [Appliance mode](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html)
- [TGW with VPN (ECMP)](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpn-attachments.html)
- [TGW quotas](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html)

---
