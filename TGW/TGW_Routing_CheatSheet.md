# AWS Transit Gateway - Routing (Route Tables, Associations & Propagations) Cheat Sheet

## Overview

TGW routing is controlled by **TGW route tables**. Each attachment is **associated** with exactly one route table (which it uses to make forwarding decisions) and can **propagate** its routes into one or more route tables (advertising its CIDRs). Segmentation is built entirely from these two knobs plus static routes.

**Key point:** **Association ≠ propagation.** Association = which table an attachment *uses*. Propagation = which tables *learn* the attachment's CIDRs. Mixing these up is the #1 TGW routing mistake.

---

## The Two Knobs

| Concept          | Answers                                          | Cardinality                    |
| ---------------- | ------------------------------------------------ | ------------------------------ |
| **Association**  | "Which route table does this attachment use to route outbound?" | Exactly **one** per attachment |
| **Propagation**  | "Which route tables should learn this attachment's CIDRs?"      | **Zero or many** route tables  |
| **Static route** | Manually add a destination → attachment          | As needed                      |

---

## How a Packet Routes

```
Packet enters TGW from attachment X
        |
        v
X's ASSOCIATED route table is consulted
        |
        ├── longest-prefix match → forward to target attachment
        ├── blackhole route      → dropped intentionally
        └── no match             → dropped
```

- Routes in a table come from **propagations** (dynamic, the attachment's CIDRs) and **static routes** (manual).
- **Longest-prefix match** decides the target, like VPC route tables.

---

## Segmentation Patterns

### Flat / any-to-any
All attachments associate + propagate to **one** route table → full mesh connectivity.

### Isolated environments (prod vs dev)
```
prod-rt:   prod VPCs associate here;  prod VPCs propagate to prod-rt
dev-rt:    dev VPCs associate here;   dev VPCs propagate to dev-rt
Result:    prod ↔ prod, dev ↔ dev, but prod ✗ dev  (no cross-propagation)
```

### Shared services
```
shared-rt: shared VPC associates here; shared VPC propagates to prod-rt AND dev-rt
prod-rt / dev-rt: prod/dev VPCs propagate to shared-rt
Result:    prod ↔ shared, dev ↔ shared, prod ✗ dev
```

### Centralized egress / inspection
```
spoke-rt:  spokes associate here; default route 0.0.0.0/0 → egress/inspection attachment (static)
egress-rt: egress VPC associates here; spokes propagate their CIDRs so return traffic finds them
```

---

## Route Types

| Route source      | How it appears                                   |
| ----------------- | ------------------------------------------------ |
| **Propagated**    | Learned automatically from an attachment's CIDRs / BGP |
| **Static**        | You add `destination → attachment` manually      |
| **Blackhole**     | Static route that drops matching traffic          |

> **TGW peering does NOT propagate routes** — for destinations across a peering, you must add **static routes** on each TGW.

---

## CLI

```bash
# Create a route table
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-0abc

# Associate an attachment with a route table (it will USE this table)
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id tgw-rtb-prod \
  --transit-gateway-attachment-id tgw-attach-vpcprod

# Enable propagation (this table LEARNS the attachment's CIDRs)
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id tgw-rtb-shared \
  --transit-gateway-attachment-id tgw-attach-vpcprod

# Static default route to an egress/inspection attachment
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-spokes \
  --destination-cidr-block 0.0.0.0/0 \
  --transit-gateway-attachment-id tgw-attach-egress

# Blackhole a range
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-spokes \
  --destination-cidr-block 10.99.0.0/16 --blackhole

# Inspect what a table knows
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id tgw-rtb-prod \
  --filters Name=state,Values=active

# Don't forget: the VPC subnet route table also needs a route to the TGW
aws ec2 create-route --route-table-id rtb-vpc \
  --destination-cidr-block 10.0.0.0/8 --transit-gateway-id tgw-0abc
```

---

## Gotchas & Caveats

1. **Association is one-per-attachment; propagation is many** — segmentation comes from *which* tables learn *which* CIDRs, not from multiple associations.
2. **Default association & propagation are ON unless you disable them at TGW creation** — leaving them on silently creates any-to-any connectivity, defeating segmentation.
3. **You must add routes on BOTH sides** — a TGW route table entry alone doesn't move traffic; the VPC's subnet route table also needs `→ tgw`.
4. **Peering routes are not propagated** — add **static routes** for peered CIDRs on each TGW; propagation stops at the peering.
5. **Longest-prefix match, not order** — a more specific route wins; a stray `/16` static can override a propagated `/24` unexpectedly.
6. **Blackhole routes cause silent drops** — great for intentional isolation, but easy to forget when debugging "why is this dropped?"
7. **Overlapping CIDRs can't be routed** — TGW does no NAT; two attachments advertising the same CIDR create ambiguity/conflict.
8. **A propagated route and a static route to the same destination** — static generally takes precedence; know your table's effective routes via `search-transit-gateway-routes`.
9. **Return path matters** — for centralized egress/inspection, the egress route table must learn spoke CIDRs (propagation) so return traffic finds its way back.
10. **Disabling propagation doesn't remove existing static routes** — clean up manually.
11. **Route limits are high but finite** (10,000 per table) — very large orgs should summarize where possible.
12. **DNS/appliance-mode interactions** — asymmetric routing across AZs still needs appliance mode even with correct route tables.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Attachments can't reach each other          | Missing propagation/static route or VPC-side route | Propagate CIDRs; add static route; add VPC→TGW route    |
| Prod can reach dev unexpectedly             | Default propagation/association left on         | Disable defaults; separate route tables; scope propagation |
| Peered TGW destinations unreachable         | Peering routes not propagated                   | Add static routes for peer CIDRs on both TGWs              |
| Traffic silently dropped                    | Blackhole route or no match                     | `search-transit-gateway-routes`; check blackhole/gaps      |
| Return traffic lost (central egress)        | Egress RT doesn't know spoke CIDRs              | Propagate spoke CIDRs into the egress route table          |
| Wrong path taken                            | Longest-prefix / static precedence              | Review effective routes; adjust specificity                |

---

## Best Practices

1. **Disable default association & propagation** at TGW creation and design tables explicitly.
2. **Use one route table per environment/segment** (prod, dev, shared, egress).
3. **Add routes on both TGW and VPC sides** as a standard checklist item.
4. **Use static routes for TGW peering** — propagation won't cross it.
5. **Propagate spoke CIDRs into egress/inspection tables** so return traffic works.
6. **Use blackhole routes** deliberately for isolation, and document them.
7. **Audit effective routes** with `search-transit-gateway-routes` before and after changes.
8. **Keep CIDRs non-overlapping** across all attachments (IPAM helps).

---

## Useful Links

- [TGW route tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Associations](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html#tgw-route-tables-association)
- [Route propagation](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html#tgw-route-tables-propagation)
- [Static routes & blackhole](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html#tgw-route-tables-static)
- [Example: isolated VPCs](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Centralized egress design](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-egress-to-internet.html)

---
