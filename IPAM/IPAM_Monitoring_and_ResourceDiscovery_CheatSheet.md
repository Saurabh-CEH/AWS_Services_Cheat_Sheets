# Amazon VPC IPAM - Monitoring & Resource Discovery Cheat Sheet

## Overview

IPAM continuously **discovers** IP resources across your accounts/Regions and **monitors utilization** of pools, VPCs, and subnets. It publishes CloudWatch metrics, detects **overlaps**, and keeps **allocation history** for audit — the visibility features that make IPAM more than just an allocator.

**Key point:** Resource discovery and org-wide monitoring are **Advanced-tier** capabilities (billed per active IP/hour). The free tier covers single-account basics.

---

## Capabilities

| Capability                 | Description                                                    |
| -------------------------- | -------------------------------------------------------------- |
| **Resource discovery**     | Finds existing VPCs/subnets/EIPs/CIDRs across accounts/Regions |
| **Utilization monitoring** | Tracks % used for pools, VPCs, subnets; CloudWatch metrics     |
| **Overlap detection**      | Flags overlapping CIDRs across the address space               |
| **Allocation history**     | Records CIDR allocation/deallocation over time (audit)         |
| **Compliance/insights**    | Shows unmanaged/overlapping/underutilized space                |
| **Public IP Insights**     | **Free** inventory/audit of all public IPv4 across accounts/Regions + unused-EIP recommendations |

---

## Public IP Insights

**Public IP Insights** is a **free** IPAM feature that gives a unified, cross-Region (and org-wide, if integrated with Organizations) view of **every public IPv4 address** your services use — so you can monitor, audit, and **cut public-IPv4 charges** (AWS bills for *all* public IPv4, including in-use and idle EIPs).

**Public IPv4 address types tracked:**

| Type                       | What it is                                                          |
| -------------------------- | ------------------------------------------------------------------- |
| **Amazon-owned EIP**       | Elastic IPs you provisioned/assigned                                |
| **EC2 public IP**          | Auto-assigned to instances in default / auto-assign subnets         |
| **BYOIP**                  | Public IPv4 you brought to AWS                                      |
| **Service-managed IP**     | Public IPv4 provisioned & managed by a service (ECS, RDS, WorkSpaces, etc.) |
| **Service-managed BYOIP**  | BYOIP managed by a service                                          |
| **Amazon-owned contiguous EIP** | EIPs allocated from an Amazon-provided contiguous public IPv4 IPAM pool |

**EIP usage breakdown** distinguishes **Associated** vs **Unassociated** (idle) EIPs/BYOIP — unassociated EIPs still incur charges, so IPAM shows a **banner recommending you release them**.

**Per-address attributes** include: association state, address type, **service** it belongs to (AGA, DMS, Redshift, RDS, ALB/NLB, NAT gateway, Site-to-Site VPN, Other), EIP ID/name, ENI ID, instance ID, security groups, public IPv4 pool ID, network border group (advertising Region), owner account, sample (last-discovery) time, and resource-discovery ID.

**Setup:** activate by creating an IPAM and integrating Public IP Insights with a **single account** or your **Organization**. It relies on resource discovery.

> **RAM permission gotcha:** a `GetIpamDiscoveredPublicAddresses` permission error on a shared resource discovery means the shared managed permission `AWSRAMPermissionIpamResourceDiscovery` needs updating to its default version by the resource-discovery owner.

---

## Monitoring

| Signal                      | Use                                                          |
| --------------------------- | ------------------------------------------------------------ |
| CloudWatch utilization metrics | Alarm before a pool/VPC/subnet exhausts                    |
| Overlap findings            | Catch conflicting CIDRs before they cause routing problems    |
| Allocation history          | `get-ipam-address-history` for audit/forensics               |
| Resource CIDRs view         | See all discovered CIDRs and their state                     |

---

## CLI

```bash
# Address history for a CIDR (who used it, when)
aws ec2 get-ipam-address-history \
  --cidr 10.0.1.0/24 --ipam-scope-id ipam-scope-0priv

# Discovered resource CIDRs
aws ec2 get-ipam-discovered-resource-cidrs \
  --ipam-resource-discovery-id ipam-res-disco-0abc --resource-region us-east-1

# Discovered accounts (org-wide discovery)
aws ec2 get-ipam-discovered-accounts \
  --ipam-resource-discovery-id ipam-res-disco-0abc --discovery-region us-east-1

# Pool utilization / allocations
aws ec2 get-ipam-pool-allocations --ipam-pool-id ipam-pool-use1

# Public IP Insights: discovered public IPv4 addresses (via a resource discovery)
aws ec2 get-ipam-discovered-public-addresses \
  --ipam-resource-discovery-id ipam-res-disco-0abc --address-region us-east-1
```

CloudWatch: IPAM publishes pool/resource utilization metrics you can alarm on (e.g., pool > 80% used).

---

## Gotchas & Caveats

1. **Resource discovery + org-wide monitoring are Advanced-tier** — billed per active managed IP/hour; free tier is single-account basics.
2. **Discovery isn't instant** — newly created resources appear after a discovery cycle; expect lag.
3. **Overlap detection is informational** — IPAM flags overlaps but does not fix routing; you still must remediate.
4. **Cost scales with monitored IP count** — large orgs with many active IPs pay more in Advanced tier; scope discovery to needed accounts/Regions.
5. **Cost allocation tags for IPAM may not appear in CUR** as expected — tag→CUR propagation has limitations; rely on IPAM metrics for utilization.
6. **Discovery needs the right IAM/Organizations setup** — org-wide discovery requires trusted access and permissions; otherwise it only sees the local account.
7. **Allocation history retention** is finite — export important history for long-term audit.
8. **Utilization metrics lag** — alarms should use sensible thresholds/periods, not expect real-time.
9. **Unmanaged CIDRs** show up as discovered but not allocated from a pool — decide whether to import them.
10. **Multiple Regions** — discovery is per operating Region; ensure the IPAM's operating Regions cover where resources live.
11. **Discovered ≠ managed** — discovery finding a CIDR doesn't put it under pool management; import if you want IPAM to track it in a pool.
12. **Threat-intel/ownership questions** (e.g., "is this AWS IP ours?") are answered via discovery + allocation history, not guesswork.
13. **Public IP Insights is FREE** (unlike Advanced-tier discovery) — use it to find and release **unassociated EIPs**, since AWS charges for all public IPv4 (in-use and idle).
14. **Public IP Insights via shared resource discovery needs the default RAM permission** — a `GetIpamDiscoveredPublicAddresses` error means `AWSRAMPermissionIpamResourceDiscovery` must be updated to its default version by the owner.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| New resources not showing                   | Discovery cycle lag / wrong Region              | Wait for discovery; check operating Regions               |
| Org-wide discovery only sees one account    | Trusted access / permissions missing            | Enable Organizations trusted access + IAM                 |
| Overlaps flagged but nothing fixed          | Detection is informational                      | Remediate routing/CIDRs manually                          |
| Higher-than-expected IPAM cost              | Advanced tier per-IP billing at scale           | Scope discovery; review tier                              |
| Cost tags missing in CUR                    | IPAM tag→CUR limitation                          | Use IPAM metrics; known limitation                        |
| Can't find who used a CIDR                  | Not checking history                            | `get-ipam-address-history`                                |

---

## Best Practices

1. **Alarm on pool/VPC/subnet utilization** (e.g., >80%) to avoid exhaustion.
2. **Use resource discovery** to inventory existing space and find overlaps/unmanaged CIDRs.
3. **Scope discovery to needed accounts/Regions** to control Advanced-tier cost.
4. **Import unmanaged CIDRs** you want IPAM to track.
5. **Use allocation history** for audits and ownership questions.
6. **Right-size the tier** — Advanced for org-wide discovery/insights; free for single-account basics.
7. **Ensure operating Regions cover all resource Regions.**
8. **Remediate overlaps** promptly since detection is only informational.

---

## Useful Links

- [Monitor CIDR usage](https://docs.aws.amazon.com/vpc/latest/ipam/monitor-cidr-compliance-ipam.html)
- [Resource discovery](https://docs.aws.amazon.com/vpc/latest/ipam/res-disc-work-with.html)
- [View IP address history](https://docs.aws.amazon.com/vpc/latest/ipam/view-history-cidr-ipam.html)
- [CloudWatch metrics for IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/monitoring-ipam.html)
- [View Public IP Insights](https://docs.aws.amazon.com/vpc/latest/ipam/view-public-ip-insights.html)
- [IPAM tiers & pricing](https://aws.amazon.com/vpc/pricing/)

---
