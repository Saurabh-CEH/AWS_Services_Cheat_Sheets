# Amazon VPC IPAM - BYOIP (Bring Your Own IP) Cheat Sheet

## Overview

**BYOIP** lets you bring your own publicly routable **IPv4 or IPv6** address ranges into AWS and manage them through IPAM's **public scope**. You provision the range (with cryptographic authorization), then AWS can **advertise** it to the internet so your resources use your own addresses.

**Key point:** BYOIP requires a valid **Route Origin Authorization (ROA)** and a **signed authorization message** proving you own the range. Advertising is a **separate, explicit** step — provisioning alone does not put the range on the internet.

---

## Core Concepts

| Concept                    | Detail                                                          |
| -------------------------- | --------------------------------------------------------------- |
| **BYOIP pool**             | A public-scope IPAM pool holding your owned CIDR                |
| **ROA**                    | Route Origin Authorization at your RIR authorizing Amazon's ASN |
| **Authorization message + signature** | Cryptographic proof of ownership provided at provisioning |
| **Advertising**            | Enabling/disabling internet advertisement of the range          |
| **CIDR size**              | Minimum sizes apply (e.g., IPv4 down to /24 for public advertisement) |

---

## Onboarding Flow

```
1. Create a ROA at your RIR authorizing Amazon's ASN (16509 / 14618 etc.) for the range
        |
2. Generate the self-signed cert + signed authorization message for the CIDR
        |
3. provision-ipam-pool-cidr with --cidr-authorization-context (message + signature)
        |
4. IPAM validates ownership → CIDR is provisioned (not yet advertised)
        |
5. Allocate addresses (e.g., to EIPs / pools) and ENABLE advertising when ready
```

---

## CLI

```bash
# Provision your CIDR into a public-scope pool with authorization context
aws ec2 provision-ipam-pool-cidr \
  --ipam-pool-id ipam-pool-public \
  --cidr 203.0.113.0/24 \
  --cidr-authorization-context Message="1|aws|<account>|203.0.113.0/24|<expiry>|SHA256|RSAPSS",Signature="<base64-signature>"

# Check provisioning state
aws ec2 get-ipam-pool-cidrs --ipam-pool-id ipam-pool-public

# Advertise / withdraw (controls internet reachability)
aws ec2 advertise-byoip-cidr --cidr 203.0.113.0/24
aws ec2 withdraw-byoip-cidr  --cidr 203.0.113.0/24
```

---

## Gotchas & Caveats

1. **ROA is mandatory and must authorize Amazon's ASN** — a missing/incorrect ROA blocks provisioning.
2. **Signed authorization message required** — the `--cidr-authorization-context` message + signature must be valid and unexpired; mistakes here are the top provisioning failure.
3. **Advertising is separate from provisioning** — the range isn't on the internet until you `advertise-byoip-cidr`; forgetting this means BYOIP addresses aren't reachable.
4. **Withdrawing advertising breaks reachability** — `withdraw-byoip-cidr` pulls the range from the internet; coordinate carefully.
5. **Minimum CIDR sizes** — public IPv4 advertisement generally requires at least a /24; smaller blocks won't be advertised globally.
6. **`Failed provision contiguous block of size`** — allocating a contiguous sub-block can fail if the pool is fragmented; request a smaller block or compact.
7. **IPv6 BYOIP has its own process** — provisioning IPv6 ranges (and advertising) differs from IPv4; follow the IPv6-specific tutorial.
8. **Deprovisioning requires withdrawing advertising and releasing allocations** first.
9. **Cross-account use** needs the BYOIP pool shared via RAM (e.g., ALB with BYOIP from a shared pool).
10. **Propagation delay** — internet advertisement changes take time to propagate via BGP; don't expect instant reachability changes.
11. **RIR/registry constraints** — some ranges/registries have specific requirements; verify eligibility before onboarding.
12. **Ongoing ROA validity** — if the ROA expires/changes, advertisement can be affected; keep it current.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Provisioning rejected                       | Bad/expired ROA or authorization signature      | Fix ROA; regenerate the signed authorization message      |
| BYOIP addresses unreachable                 | Range provisioned but not advertised            | `advertise-byoip-cidr`                                    |
| `Failed provision contiguous block`         | Pool fragmentation                               | Request a smaller block; compact/free space               |
| Cross-account ALB can't use BYOIP           | Pool not shared via RAM                           | Share the BYOIP pool via RAM to the account               |
| Advertisement change not effective          | BGP propagation delay                            | Wait for propagation; verify from external looking glass  |
| Can't deprovision                           | Still advertised / allocations present           | Withdraw advertising; release allocations first           |

---

## Best Practices

1. **Set up a correct ROA** authorizing Amazon's ASN before provisioning.
2. **Carefully generate the signed authorization message** and keep it unexpired.
3. **Provision first, advertise deliberately** when you're ready for the range to be live.
4. **Advertise at least a /24** for public IPv4 to ensure global routability.
5. **Manage BYOIP in a central public-scope pool** and share via RAM for cross-account use.
6. **Keep the ROA current** to avoid advertisement issues.
7. **Coordinate withdraw/advertise** changes to avoid outages.
8. **Watch for fragmentation** to avoid contiguous-block allocation failures.

---

## Useful Links

- [BYOIP with IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-byoip-ipam.html)
- [BYOIP IPv4 tutorial](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-byoip-ipam-ipv4.html)
- [BYOIP IPv6 tutorial](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-byoip-ipam-ipv6.html)
- [ROA and authorization](https://docs.aws.amazon.com/vpc/latest/ipam/tutorials-byoip-ipam.html)
- [advertise/withdraw BYOIP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-byoip.html)

---
