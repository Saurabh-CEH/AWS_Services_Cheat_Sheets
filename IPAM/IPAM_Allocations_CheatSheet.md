# Amazon VPC IPAM - Allocations & Auto-Allocation Cheat Sheet

## Overview

An **allocation** is a CIDR handed out from an IPAM pool — to a VPC, a subnet, or manually reserved. IPAM can **auto-allocate** a non-overlapping CIDR to a new VPC based on the pool's **allocation rules**, guaranteeing consistent, conflict-free addressing.

**Key point:** Let VPCs **auto-allocate** from IPAM pools so you never assign overlapping CIDRs by hand. Manual allocations and closed-account allocations are the main sources of "stuck" CIDRs.

---

## Allocation Types

| Type                   | How it's created                                              |
| ---------------------- | ------------------------------------------------------------- |
| **VPC allocation**     | A VPC draws its CIDR from a pool at creation (auto)           |
| **Subnet allocation**  | A subnet draws from a pool (auto, within the VPC's allocation) |
| **Manual allocation**  | You reserve a CIDR from a pool (e.g., for on-prem, future use) |
| **Discovered/imported**| Existing CIDRs found by resource discovery and imported        |

---

## How Auto-Allocation Works

```
Create VPC with --ipv4-ipam-pool-id + --ipv4-netmask-length
        |
        v
IPAM checks the pool's allocation rules (allowed netmask, required tags)
        |
        ├── rules pass + space available → allocate a non-overlapping CIDR
        └── rules fail / no space         → allocation fails
        |
        v
IPAM records the allocation; utilization updates
```

---

## CLI

```bash
# VPC auto-allocates from an IPAM pool
aws ec2 create-vpc \
  --ipv4-ipam-pool-id ipam-pool-use1 \
  --ipv4-netmask-length 16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=env,Value=prod}]'

# Manual allocation (reserve a block from the pool)
aws ec2 allocate-ipam-pool-cidr \
  --ipam-pool-id ipam-pool-use1 --netmask-length 20 \
  --description "reserved for on-prem"

# See allocations in a pool
aws ec2 get-ipam-pool-allocations --ipam-pool-id ipam-pool-use1

# Release a manual allocation
aws ec2 release-ipam-pool-allocation \
  --ipam-pool-id ipam-pool-use1 \
  --cidr 10.0.16.0/20 \
  --ipam-pool-allocation-id ipam-pool-alloc-0abc
```

---

## Gotchas & Caveats

1. **Manual allocations can linger** — a hand-reserved CIDR isn't released when the associated resource goes away; release it explicitly or the pool shows it as used.
2. **Closed-account allocations get stuck** — a CIDR allocated to a VPC in a **closed account** can be hard to disassociate; you may need a support case (a common real-world pain point).
3. **Auto-allocation obeys allocation rules** — a `--netmask-length` outside the pool's min/max, or missing required tags, fails.
4. **IPAM doesn't re-IP existing VPCs** — it tracks and allocates for new resources; migrating existing VPCs off overlapping space is still manual.
5. **Allocation ≠ advertising (public)** — allocating a public CIDR doesn't advertise it; that's a separate BYOIP step.
6. **Utilization lags slightly** — metrics/allocation views update on a schedule; recent changes may not show instantly.
7. **Subnet allocations come from the VPC's allocation** — a subnet can only draw within the VPC's IPAM-assigned space.
8. **Releasing a VPC's CIDR** requires deleting the VPC (or disassociating the CIDR) first; you can't release space still in use.
9. **Overlap protection is within a scope** — IPAM prevents overlaps in the same scope; cross-scope reuse is allowed (by design).
10. **Reclaiming space** after resource deletion may need manual release for manual allocations.
11. **Contiguous-block requirement** — some allocations need a contiguous free block; fragmentation can cause failures (see BYOIP sheet's `Failed provision contiguous block`).
12. **Cross-account allocation** requires the pool to be shared via RAM to the allocating account.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Can't disassociate CIDR (closed account)    | Allocation tied to a closed account             | Reclaim via IPAM; open a support case if stuck            |
| Auto-allocation fails                       | Netmask/tag rule violation or no space          | Fix request to match rules; free/provision space          |
| Pool shows space used but resource gone     | Lingering manual allocation                     | `release-ipam-pool-allocation`                            |
| Subnet can't get a CIDR                     | Outside VPC's IPAM allocation                   | Allocate within the VPC's assigned space                  |
| Utilization looks wrong                     | Reporting lag                                   | Wait for refresh; re-check                                |
| Cross-account VPC can't allocate            | Pool not shared to that account                 | Share the pool via RAM                                    |

---

## Best Practices

1. **Auto-allocate VPC/subnet CIDRs** from IPAM pools to guarantee no overlaps.
2. **Enforce allocation rules** (netmask ranges + required tags) for consistency.
3. **Release manual allocations** promptly when no longer needed.
4. **Reclaim CIDRs** when VPCs/accounts are decommissioned; watch for closed-account edge cases.
5. **Don't rely on IPAM to re-IP** existing VPCs — plan migrations separately.
6. **Tag allocations** for cost/ownership tracking.
7. **Share pools via RAM** for cross-account auto-allocation.
8. **Monitor utilization** to avoid exhaustion and fragmentation.

---

## Useful Links

- [Allocate CIDRs](https://docs.aws.amazon.com/vpc/latest/ipam/allocate-cidrs-ipam.html)
- [Create a VPC that uses an IPAM pool CIDR](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-allocate-vpc-ipam.html)
- [Manage allocations](https://docs.aws.amazon.com/vpc/latest/ipam/manage-cidr-ipam.html)
- [Release an allocation](https://docs.aws.amazon.com/vpc/latest/ipam/release-alloc-ipam.html)
- [Allocation rules](https://docs.aws.amazon.com/vpc/latest/ipam/allocation-rules-ipam.html)

---
