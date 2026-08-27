# Amazon VPC IPAM - Scopes & Pools Cheat Sheet

## Overview

IPAM organizes address space as a hierarchy: **IPAM → scopes → pools → allocations**. **Scopes** are routing domains (public vs private); **pools** are collections of CIDRs that nest to delegate space by Region, account, or environment.

**Key point:** A pool has a **locale** (Region) — a VPC can only draw a CIDR from a pool whose locale matches the VPC's Region. Design the hierarchy to mirror your org and Regions.

---

## Scopes

| Scope         | Purpose                                                         |
| ------------- | --------------------------------------------------------------- |
| **Private**   | RFC1918 / private routing domain (default private scope)         |
| **Public**    | Public/BYOIP address space; advertising control                  |
| **Custom**    | Additional scopes for overlapping/segregated routing domains     |

> Scopes isolate routing domains so overlapping CIDRs in different scopes don't conflict (e.g., isolated environments that intentionally reuse ranges).

---

## Pool Hierarchy

```
IPAM
 └── Private scope
       └── Top-level pool (10.0.0.0/8)           locale: none (parent)
             ├── Regional pool us-east-1 (10.0.0.0/12)   locale: us-east-1
             │     ├── Prod pool (10.0.0.0/14)
             │     └── Dev pool  (10.0.4.0/14)
             └── Regional pool eu-west-1 (10.16.0.0/12)  locale: eu-west-1
```

- **Top-level pool:** the big block; usually no locale.
- **Regional pools:** locale = a Region; VPCs in that Region allocate here.
- **Sub-pools:** per account/environment for delegation.

---

## Pool Settings

| Setting                    | Detail                                                          |
| -------------------------- | --------------------------------------------------------------- |
| **Address family**         | IPv4 or IPv6 (a pool is one family)                             |
| **Locale**                 | Region the pool serves (required for VPC auto-allocation)       |
| **Provisioned CIDRs**      | The ranges you add to the pool                                  |
| **Allocation rules**       | Allowed netmask lengths (min/max/default), required tags        |
| **Auto-import**            | Whether to auto-import discovered CIDRs into the pool           |
| **Advertisable** (public)  | Whether the range can be advertised to the internet            |

---

## CLI

```bash
# Create a private-scope top-level pool and provision a CIDR
aws ec2 create-ipam-pool \
  --ipam-scope-id ipam-scope-0priv --address-family ipv4
aws ec2 provision-ipam-pool-cidr --ipam-pool-id ipam-pool-top --cidr 10.0.0.0/8

# Create a regional sub-pool (locale = Region) with allocation rules
aws ec2 create-ipam-pool \
  --ipam-scope-id ipam-scope-0priv --address-family ipv4 \
  --source-ipam-pool-id ipam-pool-top --locale us-east-1 \
  --allocation-min-netmask-length 16 --allocation-max-netmask-length 24 \
  --allocation-default-netmask-length 20
aws ec2 provision-ipam-pool-cidr --ipam-pool-id ipam-pool-use1 --cidr 10.0.0.0/12

# Inspect
aws ec2 describe-ipam-pools
aws ec2 get-ipam-pool-cidrs --ipam-pool-id ipam-pool-use1
```

---

## Gotchas & Caveats

1. **Pool locale must match the VPC's Region** — a VPC can't allocate from a pool whose locale is a different Region (or no locale, for direct VPC allocation).
2. **A pool is a single address family** — separate pools for IPv4 and IPv6.
3. **Sub-pool CIDRs must come from the parent** — you provision the parent, then carve sub-pools from it; you can't add unrelated ranges to a sub-pool.
4. **Allocation rules constrain sizing** — min/max/default netmask and required tags; a VPC request violating them fails.
5. **Provisioning must precede allocation** — a pool with no provisioned CIDR can't hand out space.
6. **Deleting a pool needs allocations released and sub-pools removed** first.
7. **Scopes isolate routing domains** — overlapping CIDRs are only "OK" across different scopes; within a scope they still conflict.
8. **Custom scopes** are for advanced isolation — don't overuse; most orgs need just private + public.
9. **Auto-import can pull in unexpected CIDRs** — understand the setting before enabling on a pool.
10. **Public-scope pools have advertising controls** — provisioning doesn't advertise; that's a separate BYOIP step.
11. **Home Region** of the IPAM governs some management operations — pick it deliberately.
12. **Tag-based allocation rules** require the requester to set tags — missing tags block allocation.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| VPC can't allocate from a pool              | Locale mismatch or no provisioned space         | Use a pool with matching locale; provision CIDRs          |
| Allocation rejected                         | Violates netmask/tag allocation rules           | Adjust request or pool rules                              |
| Can't create sub-pool CIDR                  | Range not within parent                         | Carve sub-pool CIDR from the parent's provisioned space   |
| Pool won't delete                           | Outstanding allocations / sub-pools             | Release allocations; delete sub-pools first               |
| Overlapping CIDRs flagged                   | Same scope reuse                                | Use separate scopes or non-overlapping ranges             |
| IPv6 VPC can't use pool                     | Pool is IPv4                                    | Create/use an IPv6 pool                                   |

---

## Best Practices

1. **Mirror your org/Region structure** in the pool hierarchy (top → Region → account/env).
2. **Set locales on regional pools** so VPC auto-allocation works.
3. **Define allocation rules** (netmask ranges, required tags) to keep sizing consistent.
4. **Separate IPv4 and IPv6 pools.**
5. **Use scopes for genuine routing-domain isolation**, not as a general grouping tool.
6. **Provision generously at the top**, delegate down to sub-pools.
7. **Pick the IPAM home Region** intentionally.
8. **Document the hierarchy** so teams know which pool to draw from.

---

## Useful Links

- [How IPAM works (scopes & pools)](https://docs.aws.amazon.com/vpc/latest/ipam/how-it-works-ipam.html)
- [Create top-level pools](https://docs.aws.amazon.com/vpc/latest/ipam/create-top-ipam.html)
- [Plan for IP address provisioning](https://docs.aws.amazon.com/vpc/latest/ipam/planning-ipam.html)
- [Allocation rules](https://docs.aws.amazon.com/vpc/latest/ipam/allocation-rules-ipam.html)
- [Scopes](https://docs.aws.amazon.com/vpc/latest/ipam/how-it-works-ipam.html)

---
