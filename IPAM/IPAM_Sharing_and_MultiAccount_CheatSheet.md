# Amazon VPC IPAM - Sharing & Multi-Account Cheat Sheet

## Overview

In an Organization, IPAM is typically run from a **delegated IPAM account** and its **pools are shared via AWS RAM** to workload accounts. Those accounts then **auto-allocate** VPC CIDRs from the shared pools, guaranteeing non-overlapping addressing org-wide.

**Key point:** Sharing an IPAM **pool** (not the IPAM itself) via RAM is what lets other accounts allocate from it. Org-wide **resource discovery** additionally needs Organizations integration/delegation.

---

## Multi-Account Model

| Piece                        | Role                                                          |
| ---------------------------- | ------------------------------------------------------------- |
| **Delegated IPAM account**   | Owns the IPAM, scopes, and pool hierarchy                     |
| **RAM-shared pools**         | Pools shared to workload accounts/OUs for allocation           |
| **Workload accounts**        | Create VPCs that auto-allocate from shared pools               |
| **Organizations integration** | Enables org-wide resource discovery and delegated admin       |

---

## How It Works

```
Management account: enable IPAM integration with Organizations → delegate IPAM admin
        |
        v
IPAM (delegated account): build scopes + pool hierarchy
        |
        v
Share regional/sub-pools via RAM to OUs / accounts
        |
        v
Workload account: create VPC with --ipv4-ipam-pool-id (shared pool) → auto-allocates non-overlapping CIDR
```

---

## CLI

```bash
# (Management account) delegate IPAM admin
aws ec2 enable-ipam-organization-admin-account --delegated-admin-account-id 111122223333

# (IPAM account) share a pool via RAM to an OU
aws ram create-resource-share \
  --name "ipam-pool-share" \
  --resource-arns arn:aws:ec2::111122223333:ipam-pool/ipam-pool-use1 \
  --principals ou-abcd-12345678

# (Workload account) create a VPC drawing from the shared pool
aws ec2 create-vpc --ipv4-ipam-pool-id ipam-pool-use1 --ipv4-netmask-length 16
```

---

## Gotchas & Caveats

1. **Share the POOL, not the IPAM** — RAM sharing targets pools; workload accounts allocate from shared pools, they don't manage the IPAM.
2. **RAM sharing to OUs needs Organizations trusted access for RAM** — otherwise share by explicit account IDs.
3. **Org-wide resource discovery needs IPAM–Organizations integration** and a delegated admin — without it, discovery only sees the local account.
4. **Locale still applies to shared pools** — a workload account can only allocate from a shared pool whose locale matches the VPC's Region.
5. **Allocation rules travel with the pool** — shared pools enforce their netmask/tag rules on consumer allocations.
6. **Closed-account allocations** from shared pools can be hard to reclaim — the CIDR stays allocated; may need support.
7. **Removing a RAM share** doesn't release existing allocations — consumers keep their CIDRs; reclaim explicitly.
8. **Delegated admin is one account** — choose a stable networking account; moving it later is disruptive.
9. **Cross-account visibility is limited** — consumers see the shared pool but not the full IPAM hierarchy/management.
10. **BYOIP pools shared via RAM** enable cross-account use (e.g., ALB with BYOIP from a shared pool) — but advertising is still controlled centrally.
11. **Cost (Advanced tier)** scales with monitored IPs across the org — scope discovery to needed accounts/Regions.
12. **Consistent tagging** across accounts helps cost allocation and cleanup, since IPAM tag→CUR has limitations.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Workload account can't allocate             | Pool not shared / wrong locale                  | Share the pool via RAM; use a matching-locale pool        |
| Share to OU fails                           | RAM Organizations trusted access disabled       | Enable RAM sharing with Organizations                     |
| Org discovery sees only one account         | No IPAM–Organizations integration / delegation  | Enable integration; set delegated admin                   |
| Can't reclaim CIDR from closed account      | Allocation tied to closed account               | Reclaim via IPAM; support case if stuck                   |
| Consumer allocation violates sizing         | Pool allocation rules enforced                  | Match netmask/tag rules                                   |
| Orphaned allocations after unsharing        | Unsharing doesn't release allocations           | Release allocations explicitly                            |

---

## Best Practices

1. **Run IPAM in a delegated networking account**; enable Organizations integration.
2. **Share regional/sub-pools via RAM** to the OUs/accounts that need them.
3. **Let workload accounts auto-allocate** from shared pools for guaranteed non-overlap.
4. **Keep allocation rules on shared pools** to enforce consistent sizing/tags.
5. **Reclaim allocations** when accounts/VPCs are decommissioned; watch closed-account cases.
6. **Scope org-wide discovery** to needed accounts/Regions to manage Advanced-tier cost.
7. **Standardize tags** across accounts for cost/ownership.
8. **Pick a stable delegated admin account** up front.

---

## Useful Links

- [Share an IPAM pool using RAM](https://docs.aws.amazon.com/vpc/latest/ipam/share-pool-ipam.html)
- [Integrate IPAM with Organizations](https://docs.aws.amazon.com/vpc/latest/ipam/enable-integ-ipam.html)
- [IPAM delegated admin](https://docs.aws.amazon.com/vpc/latest/ipam/enable-integ-ipam.html)
- [AWS RAM](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html)
- [Allocate from a shared pool](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-allocate-vpc-ipam.html)

---
