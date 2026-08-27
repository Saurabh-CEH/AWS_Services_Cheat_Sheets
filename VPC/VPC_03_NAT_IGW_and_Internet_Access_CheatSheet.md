# Amazon VPC - NAT, Internet Gateway & Egress Cheat Sheet

## Overview

This sheet covers how VPC resources reach the internet: the **Internet Gateway (IGW)** for public two-way IPv4/IPv6, **NAT Gateway** for outbound-only IPv4 from private subnets, and the **Egress-only Internet Gateway** for outbound-only IPv6.

**Key point:** A NAT Gateway must sit in a **public** subnet (with an IGW route) and private subnets route `0.0.0.0/0` to the NAT. NAT is IPv4-only.

---

## Gateway Types

| Gateway                        | Direction        | IP version | Placement                       |
| ------------------------------ | ---------------- | ---------- | ------------------------------- |
| **Internet Gateway (IGW)**     | Inbound + outbound | IPv4 & IPv6 | Attached to the VPC (one per VPC) |
| **NAT Gateway**                | Outbound only    | IPv4       | In a **public** subnet, per AZ  |
| **Egress-only IGW (EIGW)**     | Outbound only    | IPv6       | Attached to the VPC             |
| **NAT Instance** (legacy)      | Outbound only    | IPv4       | Self-managed EC2 (avoid)        |

---

## Public Internet Access Requirements (IPv4)

For an instance to be reachable/able to reach the internet on IPv4, **all** must be true:

1. VPC has an **IGW** attached.
2. Subnet route table has `0.0.0.0/0 → igw-...`.
3. Instance has a **public IPv4** (auto-assign or Elastic IP).
4. **Security group** allows the traffic.
5. **NACL** allows the traffic (both directions incl. ephemeral).

> Miss any one and connectivity fails — this checklist is the fastest triage path.

---

## NAT Gateway

Lets instances in **private** subnets initiate outbound IPv4 (updates, API calls) while blocking unsolicited inbound.

| Property             | Detail                                                             |
| -------------------- | ------------------------------------------------------------------ |
| Type                 | **Public** (needs EIP, reaches internet) or **Private** (to on-prem/TGW, no EIP) |
| Placement            | One per AZ, in a **public** subnet (for public NAT)                |
| Bandwidth            | Scales automatically (up to tens of Gbps)                          |
| Elastic IP           | Public NAT needs one EIP                                            |
| Availability         | AZ-scoped — **not** cross-AZ resilient by itself                   |

### Recommended HA pattern

```
Per AZ:
  Public subnet  → NAT Gateway (with EIP)
  Private subnet → route 0.0.0.0/0 → the NAT Gateway IN THE SAME AZ
```

> Put a NAT Gateway in **each AZ** and route each private subnet to its **same-AZ** NAT. Routing across AZs adds cross-AZ data charges and creates an AZ-failure dependency.

```bash
# Allocate EIP, create public NAT Gateway
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id subnet-public-1a --allocation-id eipalloc-0abc

# Route private subnet to the NAT
aws ec2 create-route --route-table-id rtb-private-1a \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0abc
```

---

## Egress-only Internet Gateway (IPv6)

- IPv6 addresses are **globally routable** — there is no NAT for IPv6.
- To give private IPv6 instances outbound-only access, use an **EIGW** and route `::/0 → eigw-...`.

```bash
aws ec2 create-egress-only-internet-gateway --vpc-id vpc-0abc
aws ec2 create-route --route-table-id rtb-private \
  --destination-ipv6-cidr-block ::/0 --egress-only-internet-gateway-id eigw-0abc
```

---

## How Outbound Flows

```
Private instance (10.0.2.x) → route table: 0.0.0.0/0 → NAT Gateway (same AZ)
        |
        v
NAT Gateway (public subnet) → source-NAT to its EIP → IGW → internet
        |
        v
Response returns to NAT's EIP → NAT translates back → private instance
```

---

## Pricing

| Item                          | Cost driver                                                   |
| ----------------------------- | ------------------------------------------------------------- |
| Internet Gateway              | No charge for the gateway itself (data transfer applies)      |
| **NAT Gateway**               | **Hourly charge per NAT GW + per-GB data processed**          |
| Egress-only IGW               | No hourly charge (data transfer applies)                      |
| Elastic IP / public IPv4      | Hourly charge per public IPv4 (attached or idle)              |
| Data transfer out to internet | Per-GB egress charges                                          |

> NAT Gateway's **per-GB processing fee is a common cost surprise** — high-volume egress (e.g., to S3) through NAT is expensive. Use a **gateway VPC endpoint for S3/DynamoDB** to bypass NAT entirely.

---

## Quotas

| Resource                              | Default limit |
| ------------------------------------- | ------------- |
| NAT Gateways per AZ                    | 5             |
| Internet Gateways per Region          | 5 (matches VPCs) |
| Elastic IPs per Region                | 5 (adjustable) |

---

## Gotchas & Caveats

1. **NAT Gateway must be in a public subnet** (with an IGW route) — putting it in a private subnet means it has no path out.
2. **NAT is AZ-scoped, not HA on its own** — a single NAT is a single point of failure for its AZ; deploy one per AZ.
3. **Routing private subnets to a NAT in another AZ** works but adds cross-AZ charges and breaks if that AZ fails.
4. **NAT is IPv4-only** — for IPv6 outbound use an Egress-only IGW; there is no IPv6 NAT.
5. **NAT per-GB processing cost** stacks on top of data transfer — S3/DynamoDB traffic through NAT is wasteful; use gateway endpoints.
6. **NAT Gateway doesn't allow inbound-initiated connections** — it's outbound-only; use an ALB/NLB or public IP for inbound.
7. **Public IPv4 is billed hourly even when idle** — unused EIPs cost money; release them.
8. **Auto-assign public IP is a subnet setting** — an instance in a "public" subnet without it (and without an EIP) still can't reach the internet.
9. **Port allocation errors on NAT** (`ErrorPortAllocation`) occur under very high concurrent connections to the same destination — spread destinations or add NAT capacity.
10. **Deleting an IGW fails if resources still have public IPs/EIPs** routed through it — detach dependencies first.
11. **NAT Gateway idle timeout** (~350s) can drop long-lived idle connections — enable TCP keepalive.
12. **A VPC can have only one IGW attached** — you can't multi-home a VPC to two IGWs.

---

## Troubleshooting

| Issue                                        | Cause                                          | Fix                                                       |
| -------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Private instance can't reach internet        | No/incorrect NAT route; NAT in wrong subnet    | Route `0.0.0.0/0 → nat`; NAT in public subnet with EIP    |
| Public instance no internet                  | Missing public IP or IGW route                 | Assign public IP/EIP; add `0.0.0.0/0 → igw`               |
| High NAT bill                                | S3/other bulk traffic through NAT              | Add gateway endpoint for S3/DynamoDB; review egress       |
| `ErrorPortAllocation` on NAT                 | Too many simultaneous connections to one dest  | Distribute destinations; scale out; consider multiple NAT |
| Long-lived connection drops                  | NAT idle timeout                               | Enable TCP keepalive < 350s                               |
| IPv6 instance can't reach internet outbound  | No EIGW route                                  | Add `::/0 → eigw`                                          |
| Can't delete IGW                             | Public IPs/EIPs still in use                   | Release/detach dependent public IPs first                 |

---

## Best Practices

1. **One NAT Gateway per AZ**, route each private subnet to its same-AZ NAT.
2. **Use gateway endpoints for S3/DynamoDB** to avoid NAT data-processing charges.
3. **Use interface endpoints** for other AWS APIs to keep traffic off NAT/IGW.
4. **Egress-only IGW for IPv6** private outbound.
5. **Release idle EIPs / public IPs** to avoid hourly charges.
6. **Keep NAT and its route in the same AZ** to avoid cross-AZ cost and failure coupling.
7. **Monitor NAT metrics** (`BytesOutToDestination`, `ErrorPortAllocation`, `PacketsDropCount`).
8. **Prefer NAT Gateway over NAT instances** — managed, scalable, HA-capable.

---

## Useful Links

- [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [NAT gateway pricing & data processing](https://aws.amazon.com/vpc/pricing/)
- [Egress-only internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)
- [Enable internet access](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-igw-internet-access.html)
- [NAT gateway troubleshooting](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html)

---
