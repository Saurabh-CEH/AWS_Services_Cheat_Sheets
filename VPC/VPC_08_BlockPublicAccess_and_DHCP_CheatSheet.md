# AWS VPC - Block Public Access (BPA) & DHCP Option Sets Cheat Sheet

## Overview

Two account/VPC-level configuration features:

- **VPC Block Public Access (BPA)** — a centralized, authoritative switch that blocks internet access (via IGW/egress-only IGW) for VPCs/subnets in a Region, regardless of SGs/NACLs/routes.
- **DHCP option sets** — how a VPC hands out DNS servers, domain name, NTP servers, and NetBIOS settings to instances via DHCP.

---

## Part 1: VPC Block Public Access (BPA)

**VPC BPA** prevents resources in the VPCs/subnets you own in a Region from **reaching or being reached from the internet** through **internet gateways (IGW)** and **egress-only IGWs** — it **supersedes** existing VPC settings and drops traffic that would otherwise be exposed.

### Modes

| Mode                    | Effect                                                                       |
| ----------------------- | ---------------------------------------------------------------------------- |
| **off**                 | BPA not enabled; IGW/EIGW traffic allowed normally                           |
| **block-bidirectional** | Block **all** traffic to and from IGWs/EIGWs in the Region (except exclusions) |
| **block-ingress**       | Block **inbound** internet traffic; still allow outbound via **NAT gateway / egress-only IGW** (which only permit outbound-initiated connections) |

### Exclusions

- An **exclusion** is a mode applied to a **single VPC or subnet** that exempts it from the account's BPA mode, allowing **bidirectional** or **egress-only** access.
- If BPA is enabled at the **Organization** level, **exclusions may be not-allowed** (locked down); otherwise exclusions are allowed.

### Important behaviors

- **Global Accelerator:** inbound to VPCs is blocked whether or not the target is otherwise internet-accessible.
- **Network Firewall:** with BPA on, **all** inbound/outbound is blocked **even if** the firewall-endpoint subnet is excluded — plan firewall/egress designs around this.
- BPA is **Region-scoped** and applies to VPCs/subnets **you own**.
- Assess impact first with **Network Access Analyzer** (IGW ingress/egress findings show what BPA would restrict).

### Gotchas & Caveats (BPA)

1. **BPA overrides SGs/NACLs/routes** — it's authoritative; a permissive SG won't re-open blocked internet paths.
2. **`block-ingress` still allows NAT/EIGW egress** — use it for "no inbound, outbound OK"; use `block-bidirectional` for full isolation.
3. **Org-level BPA can forbid exclusions** — member accounts may be unable to self-exempt.
4. **Network Firewall + BPA**: blocked even if the firewall subnet is excluded — a common surprise.
5. **Gateway endpoints (S3/DynamoDB) aren't internet** — BPA targets IGW/EIGW paths, not PrivateLink/gateway endpoints.
6. **Assess before enabling** — use Network Access Analyzer findings to see what will break.

```bash
# View current BPA options
aws ec2 describe-vpc-block-public-access-options

# Enable block-ingress (allow outbound via NAT/EIGW)
aws ec2 modify-vpc-block-public-access-options --internet-gateway-block-mode block-ingress

# Create an exclusion for a specific VPC/subnet
aws ec2 create-vpc-block-public-access-exclusion \
  --vpc-id vpc-0abc --internet-gateway-exclusion-mode allow-bidirectional
```

---

## Part 2: DHCP Option Sets

A **DHCP option set** defines the DHCP configuration a VPC passes to instances at launch/renewal.

| Option                   | Purpose                                                              |
| ------------------------ | -------------------------------------------------------------------- |
| **domain-name-servers**  | DNS resolvers — `AmazonProvidedDNS` (VPC+2) or up to 4 custom IPs     |
| **domain-name**          | Domain suffix for hostnames                                          |
| **ntp-servers**          | NTP servers (or use the link-local time source)                     |
| **netbios-name-servers** | NetBIOS name servers                                                 |
| **netbios-node-type**    | NetBIOS node type (2 recommended)                                   |

### Key behaviors

- A VPC has **one** DHCP option set at a time; changing it **replaces** the whole set (you can't edit an existing set in place — create a new one and associate it).
- Option sets are **immutable** — to change values, create a new set and associate it to the VPC.
- New/renewed leases pick up changes; **existing instances** may need a DHCP renew/reboot to apply.
- **`AmazonProvidedDNS` is required for Route 53 Resolver / DNS Firewall to inspect queries** — if you point DNS at custom/on-prem/`8.8.8.8` resolvers via the option set, those queries **bypass** the VPC Resolver (and DNS Firewall).
- Custom `domain-name-servers` are used by instances; hardcoded resolvers in `/etc/resolv.conf` also bypass the Resolver.

### Gotchas & Caveats (DHCP)

1. **Option sets are immutable** — "editing" means create-new + re-associate.
2. **Custom DNS breaks DNS Firewall / Resolver inspection** — only queries to `AmazonProvidedDNS` are inspected/forwarded by Route 53 Resolver.
3. **Changes aren't instant on running instances** — renew the lease or reboot to apply new DNS/NTP.
4. **Only up to 4 DNS servers**; order/health matters for resolution behavior.
5. **Cross-account DNS setups** often hinge on the option set + resolver rules — a frequent source of "can't resolve internal names."
6. **Deleting an option set in use** isn't allowed — disassociate first (VPC falls back to default or another set).

```bash
# Create a DHCP option set with custom DNS + domain
aws ec2 create-dhcp-options --dhcp-configurations \
  'Key=domain-name-servers,Values=10.0.0.2' \
  'Key=domain-name,Values=example.internal'

# Associate it with a VPC (replaces the current set)
aws ec2 associate-dhcp-options --dhcp-options-id dopt-0abc --vpc-id vpc-0abc

# Revert a VPC to AmazonProvidedDNS (default)
aws ec2 associate-dhcp-options --dhcp-options-id default --vpc-id vpc-0abc
```

---

## Useful Links

- [VPC Block Public Access](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa.html)
- [VPC BPA basics / modes](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa-basics.html)
- [Assess impact of VPC BPA (Network Access Analyzer)](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa-assess-impact-main.html)
- [DHCP option sets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_DHCP_Options.html)
- [Working with DHCP option sets](https://docs.aws.amazon.com/vpc/latest/userguide/DHCPOptionSet.html)

---

*Sources: AWS official documentation. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
