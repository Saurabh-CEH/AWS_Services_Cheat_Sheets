# Amazon VPC - Core, Subnets & Routing Cheat Sheet

## Overview

An Amazon **VPC (Virtual Private Cloud)** is a logically isolated virtual network in an AWS Region where you launch resources. You control its IP addressing (CIDR blocks), subnets, route tables, and gateways.

**Key point:** A VPC is **Region-scoped** and spans all Availability Zones in that Region. A **subnet is AZ-scoped** — it lives in exactly one AZ.

---

## Core Concepts

| Component            | Description                                                                  |
| -------------------- | ---------------------------------------------------------------------------- |
| **VPC**              | Isolated virtual network; has one or more CIDR blocks                        |
| **CIDR block**       | The IP range(s) of the VPC (IPv4 and/or IPv6)                                |
| **Subnet**           | A CIDR subset within one AZ; public, private, or isolated                    |
| **Route table**      | Rules directing traffic by destination CIDR to a target                      |
| **Main route table** | Default route table for subnets with no explicit association                 |
| **Internet Gateway (IGW)** | Enables IPv4/IPv6 internet access for public subnets                   |
| **NAT Gateway**      | Outbound-only internet for private subnets (IPv4)                            |
| **Egress-only IGW**  | Outbound-only internet for IPv6                                              |
| **ENI**              | Elastic Network Interface — a virtual NIC attached to resources              |
| **DHCP option set**  | DNS/NTP/domain settings handed to instances                                  |

---

## Subnet Types

| Type        | Definition                                                            |
| ----------- | --------------------------------------------------------------------- |
| **Public**  | Route table has a route `0.0.0.0/0 → IGW`; resources can have public IPs |
| **Private** | No direct IGW route; outbound via NAT Gateway                          |
| **Isolated**| No internet route at all; reaches AWS services via VPC endpoints only  |

> "Public" vs "private" is determined **only** by the route table — there is no subnet "type" attribute.

---

## IP Addressing

| Item                       | Notes                                                             |
| -------------------------- | ----------------------------------------------------------------- |
| **Primary IPv4 CIDR**      | /16 to /28; cannot be changed after creation                      |
| **Secondary CIDRs**        | Add more IPv4 blocks later (with constraints)                     |
| **IPv6**                   | Amazon-provided /56, or bring-your-own (BYOIP) via IPAM           |
| **Reserved IPs per subnet**| AWS reserves the **first 4 and the last** address in every subnet |

### Reserved addresses in a subnet (e.g., 10.0.0.0/24)

| Address     | Reserved for                          |
| ----------- | ------------------------------------- |
| `.0`        | Network address                       |
| `.1`        | VPC router                            |
| `.2`        | Amazon DNS (base + 2)                 |
| `.3`        | Reserved for future use               |
| `.255`      | Network broadcast (not supported, reserved) |

> A /28 (16 addresses) yields only **11 usable** IPs because 5 are reserved. This is the smallest allowed subnet.

---

## How Routing Works

```
Packet leaves an ENI in a subnet
        |
        v
Subnet's associated route table evaluated — MOST SPECIFIC prefix wins (longest-prefix match)
        |
        ├── matches local VPC CIDR      → routed within VPC (implicit "local" route)
        ├── matches 0.0.0.0/0 → IGW      → internet (public subnet)
        ├── matches 0.0.0.0/0 → NAT GW   → outbound internet (private subnet)
        ├── matches peer/endpoint prefix → peering / gateway endpoint / etc.
        └── no match                     → dropped
```

- Every route table has an un-removable **`local` route** covering the VPC CIDR(s).
- **Longest-prefix match** decides the target; `10.0.1.0/24` beats `10.0.0.0/16` beats `0.0.0.0/0`.
- A subnet uses the **main route table** unless explicitly associated with another.

---

## Route Targets

| Target                     | Use                                             |
| -------------------------- | ----------------------------------------------- |
| `local`                    | Intra-VPC (implicit)                             |
| Internet Gateway (igw-)    | Internet (public)                               |
| NAT Gateway (nat-)         | Outbound IPv4 from private subnets              |
| Egress-only IGW (eigw-)    | Outbound IPv6                                    |
| VPC Peering (pcx-)         | To a peered VPC's CIDR                           |
| Gateway VPC endpoint (vpce-) | S3 / DynamoDB via prefix list                 |
| Network interface (eni-)   | Appliance/firewall routing                       |
| Gateway Load Balancer endpoint (gwlbe-) | Inline appliance inspection        |

---

## CLI Commands

```bash
# Create a VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'

# Add a secondary CIDR
aws ec2 associate-vpc-cidr-block --vpc-id vpc-0abc --cidr-block 10.1.0.0/16

# Create subnets in specific AZs
aws ec2 create-subnet --vpc-id vpc-0abc --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-0abc --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b

# Create a route table and a default route to an IGW
aws ec2 create-route-table --vpc-id vpc-0abc
aws ec2 create-route --route-table-id rtb-0abc \
  --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0abc

# Associate a subnet with a route table
aws ec2 associate-route-table --route-table-id rtb-0abc --subnet-id subnet-0abc

# Enable DNS hostnames / support (needed for Resolver, endpoints, PHZ)
aws ec2 modify-vpc-attribute --vpc-id vpc-0abc --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-0abc --enable-dns-support

# Auto-assign public IPv4 on a public subnet
aws ec2 modify-subnet-attribute --subnet-id subnet-0abc --map-public-ip-on-launch

# Inspect
aws ec2 describe-vpcs
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-0abc
aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-0abc
```

---

## Pricing

| Item                              | Cost                                              |
| --------------------------------- | ------------------------------------------------- |
| VPC, subnets, route tables, IGW   | No charge                                          |
| Data transfer within same AZ      | Free (private IPs)                                 |
| Cross-AZ data transfer            | Charged per GB each direction                      |
| NAT Gateway                       | Hourly + per-GB processed (see NAT/IGW sheet)      |
| Public IPv4 addresses             | Hourly charge per public IPv4 (in-use and idle)    |

> Since 2024, **all public IPv4 addresses are billed hourly** whether attached or idle — audit unused EIPs and public IPs.

---

## Quotas (defaults, adjustable)

| Resource                                | Default limit         |
| --------------------------------------- | --------------------- |
| VPCs per Region                         | 5                     |
| IPv4 CIDR blocks per VPC                 | 5 (up to 50)          |
| IPv6 CIDR blocks per VPC                 | 5 (up to 50)          |
| Subnets per VPC                          | 200                   |
| Route tables per VPC                     | 200                   |
| Routes per route table (non-propagated) | 50 (up to 1,000)      |
| Security groups per VPC                  | 2,500                 |

---

## Gotchas & Caveats

1. **You cannot change or shrink the primary VPC CIDR** after creation — only add/remove secondary CIDRs (with restrictions). Plan addressing up front.
2. **5 IPs per subnet are reserved** (first 4 + last) — a /28 gives 11 usable, not 16, and /28 is the smallest allowed subnet.
3. **A subnet lives in exactly one AZ** — for HA, create a subnet per AZ and spread resources.
4. **"Public subnet" is purely a routing property** — auto-assign public IP + an IGW route make it public; there's no type flag.
5. **The `local` route can't be removed** and always wins for the VPC CIDR — you can't blackhole intra-VPC traffic via routes (use SGs/NACLs).
6. **Overlapping CIDRs block peering/connectivity** — VPCs with overlapping ranges can't peer or share routes; design non-overlapping space (IPAM helps).
7. **Secondary CIDR restrictions** — you can't add a range that overlaps existing routes, and some ranges (e.g., within `172.17.0.0/16` used by some services) are restricted.
8. **DNS attributes matter** — `enableDnsSupport` and `enableDnsHostnames` must be on for Resolver, private hosted zones, and many endpoints to work.
9. **Deleting a VPC requires tearing down dependencies first** — ENIs (often left by Lambda/ELB), endpoints, gateways, and peering must go first.
10. **Longest-prefix match, not order** — route tables aren't evaluated top-to-bottom; the most specific matching route wins.
11. **Default VPC quirks** — the default VPC has public subnets and auto-assign public IP on; don't assume it's private.
12. **Broadcast/multicast are not supported** in a VPC (except via Transit Gateway multicast).

---

## Troubleshooting

| Issue                                   | Cause                                          | Fix                                                     |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------- |
| Instance has no internet                | Missing IGW route / no public IP / SG/NACL      | Add `0.0.0.0/0 → igw`; assign public IP; check SG/NACL  |
| Private instance can't reach internet   | No NAT route or NAT in wrong subnet             | Route `0.0.0.0/0 → nat`; NAT must be in a public subnet |
| "Not enough free addresses" on launch   | Subnet exhausted                                | Use a larger subnet or add capacity; check reserved IPs |
| Can't delete subnet/VPC                 | ENIs or dependencies still present              | Remove ENIs/endpoints/gateways first                    |
| Cross-AZ traffic unexpectedly costly    | Resources spread across AZs                     | Keep chatty traffic same-AZ where possible              |
| Peering route not working               | Overlapping CIDRs or missing route both sides   | Ensure non-overlapping CIDRs; add routes on both VPCs   |
| DNS/endpoint features not working       | DNS attributes disabled                         | Enable `enableDnsSupport` + `enableDnsHostnames`        |

---

## Best Practices

1. **Plan CIDRs with room to grow** and keep them **non-overlapping** across VPCs/accounts (use IPAM).
2. **One subnet tier per AZ** (public/private/isolated × AZs) for HA.
3. **Use private + isolated subnets by default**; expose only what must be public.
4. **Reach AWS services privately** via gateway/interface endpoints instead of NAT where possible.
5. **Tag everything** (Name, environment, owner) for cost allocation and cleanup.
6. **Enable Flow Logs** for visibility and troubleshooting.
7. **Audit public IPv4** regularly now that idle addresses bill hourly.
8. **Automate with IaC** so networking is reproducible and reviewable.

---

## Useful Links

- [What is Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [VPC and subnet sizing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
- [Subnets for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [Route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [IP addressing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html)
- [Public IPv4 address charges](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)
- [Amazon VPC Quotas](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html)

---
