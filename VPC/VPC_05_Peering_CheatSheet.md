# Amazon VPC - VPC Peering Cheat Sheet

## Overview

A **VPC peering connection** is a private, one-to-one network link between two VPCs, using AWS's backbone. Peered VPCs route to each other's private IPs as if on the same network. Peering works **same or cross-account** and **same or cross-Region**.

**Key point:** Peering is **non-transitive** and requires **non-overlapping CIDRs**. Traffic never traverses the internet, an IGW, or a gateway.

---

## Core Concepts

| Concept                | Detail                                                              |
| ---------------------- | ------------------------------------------------------------------- |
| **Requester/Accepter** | One VPC requests, the other accepts the connection                  |
| **Non-transitive**     | If A↔B and B↔C peer, A cannot reach C through B                     |
| **Cross-Region**       | Supported (inter-Region peering) — traffic stays on AWS backbone    |
| **Cross-account**      | Supported — accepter approves and adds routes on their side         |
| **Routing**            | Each VPC adds routes to the other's CIDR, targeting `pcx-...`       |
| **DNS resolution**     | Optionally enable resolving the peer's private DNS hostnames        |

---

## How It Works

```
VPC A (10.0.0.0/16)                     VPC B (10.1.0.0/16)
  route: 10.1.0.0/16 → pcx-123   <──── peering ────►   route: 10.0.0.0/16 → pcx-123
  SG/NACL allow B's CIDR                                SG/NACL allow A's CIDR
```

Setup steps:
1. Requester creates the peering connection to the target VPC (same/cross account/Region).
2. Accepter **accepts** the request.
3. **Both** VPCs add routes to the other's CIDR pointing at `pcx-...`.
4. **Both** update SGs/NACLs to allow the peer's traffic.

---

## CLI

```bash
# Request peering (cross-account/Region supported)
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-A \
  --peer-vpc-id vpc-B \
  --peer-owner-id 222233334444 \
  --peer-region us-west-2

# Accept (run in the accepter account/Region)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-0abc

# Add routes on BOTH sides
aws ec2 create-route --route-table-id rtb-A \
  --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-0abc
aws ec2 create-route --route-table-id rtb-B \
  --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-0abc

# Enable DNS resolution across the peering (optional)
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-0abc \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
```

---

## Peering vs Transit Gateway

| Aspect            | VPC Peering                          | Transit Gateway                          |
| ----------------- | ------------------------------------ | ---------------------------------------- |
| Topology          | 1:1, full mesh needed for many VPCs  | Hub-and-spoke, scales to thousands       |
| Transitive        | **No**                               | Yes (via TGW route tables)               |
| Cost              | No hourly fee; data transfer only    | Hourly attachment + per-GB               |
| Best for          | A few VPCs, lowest cost              | Many VPCs, central routing, on-prem       |

> For more than a handful of VPCs, a full mesh of peerings becomes unmanageable (n×(n-1)/2 links) — use Transit Gateway.

---

## Pricing

| Item                            | Cost                                          |
| ------------------------------- | --------------------------------------------- |
| Peering connection              | No hourly charge                              |
| Same-Region, same-AZ traffic    | Free (private IPs)                            |
| Same-Region, cross-AZ traffic   | Per-GB each direction                         |
| Inter-Region peering traffic    | Inter-Region data transfer rates              |

---

## Quotas (defaults, adjustable)

| Resource                                     | Default limit |
| -------------------------------------------- | ------------- |
| Active peering connections per VPC           | 50 (up to 125) |
| Outstanding peering requests                 | 25            |
| Inter-Region peering                         | Supported in most Regions |

---

## Gotchas & Caveats

1. **Non-transitive** — the biggest gotcha. A↔B and B↔C does NOT give A↔C. You must peer A↔C directly (or use TGW).
2. **CIDRs must not overlap** — overlapping VPC ranges cannot peer at all.
3. **Routes are required on BOTH sides** — accepting the connection alone doesn't move traffic; each VPC needs a route to the other's CIDR.
4. **SGs/NACLs must allow the peer** — connectivity fails silently if the peer's CIDR isn't allowed.
5. **You can reference peer security groups**, but only in the **same Region** peering (not inter-Region).
6. **Edge-to-edge routing is not supported** — a peer VPC can't use your IGW, NAT, VPN, or gateway endpoints; each VPC handles its own internet/egress.
7. **No transitive access to on-prem** — a VPC peered to a VPC that has a VPN/DX cannot reach on-prem through it.
8. **Cross-Region peering doesn't support IPv6** in some cases and has jumbo-frame limits — check current constraints.
9. **DNS resolution across peering is opt-in** — enable the peering DNS option to resolve the peer's private hostnames.
10. **Placement groups don't span peering** — cluster placement group benefits don't cross the connection.
11. **MTU/jumbo frames** — same-Region peering supports up to 9001 bytes; inter-Region is limited to 1500.
12. **Deleting a VPC or the connection** leaves stale routes pointing at a blackholed `pcx-` — clean them up.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Peered instances can't reach each other     | Missing route on one/both sides                 | Add routes to peer CIDR on both route tables              |
| Traffic still blocked after routes added    | SG/NACL not allowing peer CIDR                  | Allow peer CIDR in SGs and NACLs                          |
| Can't create peering                        | Overlapping CIDRs                               | Re-IP one VPC; peering requires non-overlap               |
| A can't reach C through B                   | Peering is non-transitive                       | Peer A↔C directly or migrate to Transit Gateway           |
| Can't reach peer's internet/NAT             | Edge-to-edge routing unsupported                | Each VPC needs its own IGW/NAT                             |
| Peer hostnames won't resolve                | Peering DNS resolution disabled                 | Enable `AllowDnsResolutionFromRemoteVpc`                  |
| Can't reference peer SG                     | Inter-Region peering                            | SG referencing is same-Region only; use CIDRs             |

---

## Best Practices

1. **Design non-overlapping CIDRs** across all VPCs/accounts from day one (use IPAM).
2. **Add routes and SG/NACL allows on both sides** as part of the setup checklist.
3. **Use least-privilege routing** — route only the specific subnets/CIDRs you need, not always the whole VPC.
4. **Move to Transit Gateway** when you exceed a handful of VPCs or need transitive/on-prem routing.
5. **Enable peering DNS resolution** if apps use private hostnames.
6. **Clean up stale routes** when tearing down connections.
7. **Document the mesh** — track who peers with whom to avoid accidental exposure.

---

## Useful Links

- [What is VPC peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [Create a peering connection](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html)
- [Update route tables for peering](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html)
- [Peering limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html#vpc-peering-limitations)
- [Unsupported peering configurations](https://docs.aws.amazon.com/vpc/latest/peering/invalid-peering-configurations.html)
- [Enable DNS resolution](https://docs.aws.amazon.com/vpc/latest/peering/modify-peering-connections.html)

---
