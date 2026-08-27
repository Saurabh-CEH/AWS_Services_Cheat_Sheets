# AWS Transit Gateway - Peering Cheat Sheet

## Overview

**Transit Gateway peering** connects two Transit Gateways — in the **same or different Regions and/or accounts** — so networks attached to each TGW can communicate over the AWS backbone. It's the primary way to build **global, multi-Region** networks with TGW.

**Key point:** TGW peering is **not transitive** and **does not propagate routes** — you must add **static routes** on each TGW pointing to the peering attachment for the remote CIDRs.

---

## Core Concepts

| Concept                | Detail                                                          |
| ---------------------- | --------------------------------------------------------------- |
| **Peering attachment** | A special attachment linking two TGWs                           |
| **Requester/Accepter** | One TGW requests, the other accepts                             |
| **Cross-Region**       | Supported — traffic stays on the AWS global backbone            |
| **Cross-account**      | Supported — the peer account accepts the request                |
| **Routing**            | **Static routes only** — no dynamic propagation across peering  |
| **Encryption**         | Inter-Region peering traffic is encrypted on the backbone       |

---

## How It Works

```
TGW-A (us-east-1)                         TGW-B (eu-west-1)
  route table: 10.1.0.0/16 → peering-attach   route table: 10.0.0.0/16 → peering-attach
        \                                            /
         \────────── TGW peering (backbone) ────────/
```

Setup:
1. Create the peering attachment from TGW-A to TGW-B (specify peer Region/account).
2. **Accept** the peering on TGW-B.
3. Add **static routes** on each TGW's relevant route table for the remote CIDRs → peering attachment.
4. Ensure VPC/spoke route tables route the remote CIDRs → their local TGW.

---

## CLI

```bash
# Create peering (cross-Region / cross-account)
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id tgw-A \
  --peer-transit-gateway-id tgw-B \
  --peer-account-id 222233334444 \
  --peer-region eu-west-1

# Accept (run in the peer account/Region)
aws ec2 accept-transit-gateway-peering-attachment \
  --transit-gateway-attachment-id tgw-attach-peer

# Static routes on EACH TGW (propagation does not cross peering)
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-A \
  --destination-cidr-block 10.1.0.0/16 \
  --transit-gateway-attachment-id tgw-attach-peer

aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-B \
  --destination-cidr-block 10.0.0.0/16 \
  --transit-gateway-attachment-id tgw-attach-peer
```

---

## Pricing

| Item                          | Cost                                          |
| ----------------------------- | --------------------------------------------- |
| Peering attachment            | Hourly per attachment (on each TGW)           |
| Data processed                | Per-GB processed                              |
| Inter-Region transfer         | Inter-Region data transfer rates apply        |

---

## Quotas (defaults, adjustable)

| Resource                                     | Default limit |
| -------------------------------------------- | ------------- |
| Peering attachments per TGW                  | counts toward total attachments (5,000) |
| Peering between two TGWs                      | One peering attachment per TGW pair |

---

## Gotchas & Caveats

1. **No route propagation across peering** — you MUST add static routes on each TGW for the remote CIDRs. This is the number-one peering failure.
2. **Non-transitive** — TGW-A peered to TGW-B, and TGW-B peered to TGW-C, does NOT give A↔C. Peer A↔C directly.
3. **Static routes needed on both TGWs** — one side alone gives one-way (broken) routing.
4. **VPC/spoke route tables also need the remote CIDRs** pointing at their local TGW — TGW-side routes aren't enough.
5. **Overlapping CIDRs can't be routed** across peering — no NAT.
6. **One peering attachment per TGW pair** — you can't create multiple peerings between the same two TGWs.
7. **Multicast doesn't traverse peering** — TGW multicast is intra-TGW only.
8. **Appliance-mode/inspection across Regions** requires careful design — return traffic must come back through the same inspection path.
9. **Security group referencing doesn't span peering/Regions** — use CIDRs across peered TGWs.
10. **Accept step is required** for cross-account peering (and cross-account attachments generally).
11. **MTU** across inter-Region peering is limited (1500) vs intra-Region — jumbo frames won't traverse.
12. **Deleting a peering** leaves stale static routes on both TGWs — clean them up.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Peered networks can't communicate           | Missing static routes on one/both TGWs          | Add static routes for remote CIDRs on each TGW             |
| One-way connectivity                        | Route on only one TGW / one route table         | Add matching static routes both sides                      |
| Spokes still can't reach remote             | VPC subnet route tables lack remote CIDR        | Add remote CIDR → local TGW in VPC route tables            |
| A can't reach C via B                       | Peering is non-transitive                       | Peer A↔C directly                                          |
| Peering stuck pending                       | Accept step not done (cross-account)            | Accept the peering in the peer account                     |
| Large frames dropped inter-Region           | MTU limited to 1500                             | Avoid jumbo frames across inter-Region peering             |

---

## Best Practices

1. **Add static routes on both TGWs** and in spoke VPC route tables as a setup checklist.
2. **Peer directly** for any pair that must communicate — don't rely on transitivity.
3. **Keep CIDRs non-overlapping** globally (IPAM) so peering routing is unambiguous.
4. **Document the peering mesh** and its static routes.
5. **Design return paths** for cross-Region inspection deliberately.
6. **Clean up stale static routes** when removing peerings.
7. **Use a networking account** to own TGWs and manage cross-account peering via acceptance workflows.

---

## Useful Links

- [Transit gateway peering attachments](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)
- [Create a peering attachment](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html#tgw-peering-create)
- [Peering routing (static)](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Multi-Region design (whitepaper)](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-gateway-peering.html)

---
