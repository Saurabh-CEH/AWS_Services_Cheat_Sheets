# Amazon VPC IP Address Manager (IPAM) - Cheat Sheet

## Overview

**Amazon VPC IP Address Manager (IPAM)** is a service for planning, tracking, allocating, and monitoring IP addresses across your AWS accounts, VPCs, and Regions. It gives a hierarchical view of your address space (pools), automates CIDR allocation, detects overlaps, and integrates with **BYOIP**.

**Key point:** IPAM is organized as a hierarchy — **IPAM → scopes → pools → allocations** — and is typically shared across an **Organization** from a delegated IPAM account via RAM.

---

## Core Concepts

| Component        | Description                                                              |
| ---------------- | ------------------------------------------------------------------------ |
| **IPAM**         | The top-level resource; has a home Region and operating Regions          |
| **Scope**        | Routing domain for pools: **public** or **private** (you can add more)   |
| **Pool**         | A collection of CIDRs, hierarchical (top-level → regional → dev/prod)     |
| **Allocation**   | A CIDR handed out from a pool (to a VPC, subnet, or manually)            |
| **BYOIP**        | Bring-your-own public IPv4/IPv6 ranges into a pool                        |
| **Advertising**  | For public/BYOIP pools, whether AWS advertises the range to the internet |

---

## Scopes & Pool Hierarchy

```
IPAM
 ├── Private scope
 │     └── Top-level pool (10.0.0.0/8)
 │            ├── Regional pool us-east-1 (10.0.0.0/12)
 │            │      ├── Prod pool
 │            │      └── Dev pool
 │            └── Regional pool eu-west-1 (10.16.0.0/12)
 └── Public scope
        └── BYOIP pool (203.0.113.0/24)
```

- **Scopes** isolate routing domains (private RFC1918 space vs public/BYOIP).
- **Pools** nest so you can delegate ranges by Region/account/environment.
- VPCs can **auto-allocate** their CIDR from a pool (with allocation rules like netmask length).

---

## Key Capabilities

| Capability                     | Description                                                     |
| ------------------------------ | --------------------------------------------------------------- |
| **Automatic allocation**       | VPCs draw CIDRs from a pool per allocation rules                |
| **Overlap detection**          | Flags overlapping CIDRs across the address space                |
| **Utilization monitoring**     | Tracks pool/VPC/subnet usage; CloudWatch metrics + thresholds   |
| **Historical tracking**        | Records CIDR allocation history for audit                       |
| **Cross-account/Region**       | Share pools via RAM; operate across Regions                     |
| **BYOIP integration**          | Import and manage owned public ranges; control advertising      |
| **Resource discovery**         | Automatically finds existing VPCs/CIDRs across the org          |

---

## How Allocation Works

```
Create VPC (or subnet) requesting a CIDR from an IPAM pool
        |
        v
IPAM checks pool allocation rules (allowed netmask, tags, etc.)
        |
        ├── space available + rules pass → allocate a non-overlapping CIDR
        └── no space / rule fail         → allocation fails
        |
        v
IPAM records the allocation and updates utilization
```

---

## CLI

```bash
# Create an IPAM (in the delegated IPAM account), with operating Regions
aws ec2 create-ipam --operating-regions RegionName=us-east-1 RegionName=eu-west-1

# Create a top-level private pool and provision a CIDR
aws ec2 create-ipam-pool --ipam-scope-id ipam-scope-0abc \
  --address-family ipv4 --locale us-east-1
aws ec2 provision-ipam-pool-cidr --ipam-pool-id ipam-pool-0abc --cidr 10.0.0.0/8

# Allocate a CIDR (manual) or let a VPC auto-allocate
aws ec2 allocate-ipam-pool-cidr --ipam-pool-id ipam-pool-0abc --netmask-length 16

# Create a VPC that draws from the pool
aws ec2 create-vpc --ipv4-ipam-pool-id ipam-pool-0abc --ipv4-netmask-length 16

# BYOIP: provision your own public CIDR (needs authorization + signed message)
aws ec2 provision-ipam-pool-cidr --ipam-pool-id ipam-pool-pub \
  --cidr 203.0.113.0/24 \
  --cidr-authorization-context Message="...",Signature="..."

# Discover/inspect
aws ec2 get-ipam-pool-allocations --ipam-pool-id ipam-pool-0abc
aws ec2 get-ipam-address-history --cidr 10.0.1.0/24 --ipam-scope-id ipam-scope-0abc
```

---

## Pricing

| Item                          | Cost                                              |
| ----------------------------- | ------------------------------------------------- |
| IPAM (free tier)              | Basic features (some) at no charge                |
| **Advanced tier**             | **Per active IP address / hour** monitored across accounts/Regions |
| BYOIP                         | Standard IP charges apply                          |

> The **Advanced tier** (cross-account org-wide, resource discovery, historical insights, some monitoring) bills **per active managed IP per hour** — cost scales with the number of IPs IPAM tracks. Free tier covers single-account basics.

---

## Quotas (defaults, adjustable)

| Resource                                | Default limit |
| --------------------------------------- | ------------- |
| IPAMs per account                       | (adjustable)  |
| Pools per IPAM                          | (adjustable)  |
| Operating Regions per IPAM              | (adjustable)  |
| CIDRs per pool                          | (adjustable)  |

---

## Gotchas & Caveats

1. **Free vs Advanced tier differ significantly** — org-wide/cross-account management, resource discovery, and historical insights are **Advanced tier** (paid per IP/hour). Single-account basics are free.
2. **IPAM has a home Region** — some management operations are tied to it; plan which Region owns the IPAM.
3. **Pools are locale-scoped** — a regional pool has a `locale` (Region); a VPC can only draw from a pool whose locale matches its Region.
4. **Deleting a pool requires releasing allocations first** — you can't delete a pool with outstanding CIDRs; deprovision/reclaim first.
5. **Allocations can linger after a VPC is deleted** — especially for **closed accounts** or manual allocations; you may be unable to disassociate a CIDR tied to a closed account without support.
6. **BYOIP requires ROA + a signed authorization message** — importing public ranges needs a valid Route Origin Authorization and the cryptographic authorization context; mistakes block provisioning.
7. **BYOIP contiguous-block provisioning can fail** (`Failed provision ... contiguous block of size`) if the requested size isn't available contiguously in the pool.
8. **IPAM doesn't move existing CIDRs for you** — it tracks and can allocate, but re-IPing existing VPCs is still a manual migration.
9. **Overlap detection is informational** — IPAM flags overlaps but doesn't automatically fix routing; you still must design non-overlapping space.
10. **RAM sharing is required for cross-account pools** — share the pool via Resource Access Manager, and the consumer account allocates from it.
11. **Cost allocation tags for IPAM resources may not surface in CUR** as expected — tag propagation to Cost and Usage Reports has limitations.
12. **Advertising control for public pools is explicit** — a BYOIP range won't be advertised to the internet until you enable advertising (and stopping advertising affects reachability).

---

## Troubleshooting

| Issue                                       | Cause                                           | Fix                                                        |
| ------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| VPC can't allocate from pool                | Locale mismatch / no space / rule violation     | Use a pool matching the VPC's Region; free space; fix rules |
| `Failed provision contiguous block`         | No contiguous free block of that size           | Request a smaller block or free/compact the pool           |
| Can't disassociate CIDR (closed account)    | Allocation tied to a closed account             | Reclaim via IPAM; open a support case if stuck             |
| BYOIP provisioning rejected                 | Bad/missing ROA or authorization signature      | Fix ROA; regenerate signed authorization context           |
| Pool won't delete                           | Outstanding allocations                          | Deprovision/reclaim all allocations first                  |
| Cost tags missing in CUR                    | IPAM tag→CUR limitations                          | Known limitation; track via IPAM metrics                   |
| Unexpected IPAM cost                        | Advanced tier per-IP billing at scale           | Review tier; scope managed accounts/Regions                |

---

## Best Practices

1. **Run IPAM from a delegated networking/IPAM account** and share pools org-wide via RAM.
2. **Design a clear pool hierarchy** (top-level → Region → account/env) mirroring your org.
3. **Enforce allocation rules** (allowed netmask lengths, required tags) to keep sizing consistent.
4. **Let VPCs auto-allocate** from pools to guarantee non-overlapping CIDRs.
5. **Monitor utilization** with CloudWatch and set threshold alarms before exhaustion.
6. **Use overlap detection + resource discovery** to inventory and clean up existing space.
7. **Manage BYOIP centrally**, with correct ROAs and deliberate advertising control.
8. **Reclaim allocations** promptly when VPCs/accounts are decommissioned.
9. **Right-size the tier** — use free tier where single-account basics suffice; Advanced for org-wide.

---

## Useful Links

- [What is IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html)
- [How IPAM works](https://docs.aws.amazon.com/vpc/latest/ipam/how-it-works-ipam.html)
- [Scopes and pools](https://docs.aws.amazon.com/vpc/latest/ipam/how-it-works-ipam.html)
- [Plan for IP address provisioning](https://docs.aws.amazon.com/vpc/latest/ipam/planning-ipam.html)
- [BYOIP with IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-byoip-ipam.html)
- [IPAM tiers & pricing](https://aws.amazon.com/vpc/pricing/)
- [Share pools with RAM](https://docs.aws.amazon.com/vpc/latest/ipam/share-pool-ipam.html)

---
