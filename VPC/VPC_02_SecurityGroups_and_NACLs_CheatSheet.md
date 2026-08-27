# Amazon VPC - Security Groups & Network ACLs Cheat Sheet

## Overview

**Security Groups (SGs)** and **Network ACLs (NACLs)** are the two packet-filtering layers in a VPC. SGs are **stateful** and attach to ENIs (instance-level); NACLs are **stateless** and attach to subnets (subnet-level).

**Key point:** SGs are the primary control (stateful, allow-only). NACLs are a coarse, stateless subnet-level backstop that can also express explicit **deny**.

---

## Security Groups vs NACLs

| Aspect                | Security Group                        | Network ACL                              |
| --------------------- | ------------------------------------- | ---------------------------------------- |
| Level                 | ENI / instance                        | Subnet                                    |
| State                 | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow both directions) |
| Rules                 | **Allow only**                        | **Allow and Deny**                        |
| Evaluation            | All rules evaluated; allow if any match | Rules processed **in number order**, first match wins |
| Default (custom)      | Deny all inbound, allow all outbound  | Custom NACL denies all in/out until you add rules |
| Default (default SG/NACL) | Default SG allows intra-SG; default NACL allows all | — |
| Applies to            | Only resources it's attached to       | All resources in associated subnets       |

---

## Security Groups (Stateful)

- **Stateful:** if you allow inbound on a port, the response is automatically allowed out (and vice-versa). You don't need a matching reverse rule.
- **Allow-only:** there is no deny rule. Anything not explicitly allowed is denied.
- **Referencing:** a rule source/destination can be another **security group ID** (or the same SG) — great for tier-to-tier rules without hardcoding IPs.
- **Multiple SGs per ENI:** rules are additive (union of all allows).

### Example rules

| Direction | Protocol | Port | Source/Dest              | Purpose                    |
| --------- | -------- | ---- | ------------------------ | -------------------------- |
| Inbound   | TCP      | 443  | 0.0.0.0/0                | Public HTTPS               |
| Inbound   | TCP      | 3306 | sg-app (app tier SG)     | DB reachable only from app |
| Outbound  | All      | All  | 0.0.0.0/0                | Default allow-all egress   |

```bash
# Create SG and rules
aws ec2 create-security-group --group-name web-sg --description "web" --vpc-id vpc-0abc
aws ec2 authorize-security-group-ingress --group-id sg-web \
  --protocol tcp --port 443 --cidr 0.0.0.0/0
# Reference another SG as the source (tier-to-tier)
aws ec2 authorize-security-group-ingress --group-id sg-db \
  --protocol tcp --port 3306 --source-group sg-app
# Restrict egress (must then explicitly allow what you need)
aws ec2 revoke-security-group-egress --group-id sg-web \
  --protocol -1 --port -1 --cidr 0.0.0.0/0
```

---

## Network ACLs (Stateless)

- **Stateless:** you must allow both the inbound request AND the outbound response (and vice-versa). Return traffic uses **ephemeral ports** (typically 1024–65535).
- **Ordered, numbered rules:** evaluated low-to-high; the first match (allow or deny) wins. Best practice: leave gaps (100, 200, 300).
- **Explicit deny:** unlike SGs, NACLs can deny specific IPs/ports — useful for blocking a known-bad source.
- The final rule `*` is an implicit **deny all**.

### Example NACL

| Rule # | Type      | Protocol | Port range   | Source        | Allow/Deny |
| ------ | --------- | -------- | ------------ | ------------- | ---------- |
| 100    | Inbound   | TCP      | 443          | 0.0.0.0/0     | ALLOW      |
| 200    | Inbound   | TCP      | 1024–65535   | 0.0.0.0/0     | ALLOW (ephemeral return) |
| *      | Inbound   | All      | All          | 0.0.0.0/0     | DENY       |
| 100    | Outbound  | TCP      | 1024–65535   | 0.0.0.0/0     | ALLOW (responses) |
| *      | Outbound  | All      | All          | 0.0.0.0/0     | DENY       |

```bash
aws ec2 create-network-acl --vpc-id vpc-0abc
aws ec2 create-network-acl-entry --network-acl-id acl-0abc --rule-number 100 \
  --protocol tcp --port-range From=443,To=443 --cidr-block 0.0.0.0/0 --rule-action allow --ingress
```

---

## How a Packet Is Filtered

```
Inbound packet to an instance
        |
        v
Subnet NACL (inbound rules, numbered order, first match wins)
        |
        v
Security Group (inbound rules; stateful — allow if any match)
        |
        v
Instance
        |
   response leaves:
        |
        v
Security Group (stateful — return auto-allowed)
        |
        v
Subnet NACL (OUTBOUND rules must allow the ephemeral response)
```

---

## Ephemeral Ports (NACL gotcha)

Return traffic arrives on a client-chosen **ephemeral port**. Ranges vary by client OS:

| Client                     | Typical ephemeral range |
| -------------------------- | ----------------------- |
| Linux kernel               | 32768–60999             |
| Windows (newer)            | 49152–65535             |
| NAT Gateway                | 1024–65535              |
| Safe catch-all for NACLs   | 1024–65535              |

> Use **1024–65535** in NACL rules for return traffic to cover all clients, including NAT Gateway.

---

## Quotas (defaults, adjustable)

| Resource                              | Default limit |
| ------------------------------------- | ------------- |
| Security groups per VPC               | 2,500         |
| Inbound/outbound rules per SG         | 60 each (120 total default) |
| Security groups per ENI               | 5 (up to 16)  |
| Rules per NACL                        | 20 (up to 40) inbound + same outbound |
| NACLs per VPC                         | 200           |

> **Rules-per-SG × SGs-per-ENI is capped** (e.g., 60 rules × 5 SGs). Raising one may require lowering the other.

---

## Gotchas & Caveats

1. **SGs are stateful, NACLs are stateless** — the #1 confusion. With NACLs you must open the ephemeral return range or connections hang.
2. **SGs have no deny rule** — to block a specific bad IP you need a NACL deny (or WAF/firewall). SGs can only allow.
3. **NACL rules are order-dependent; SG rules are not** — a low-numbered NACL deny short-circuits later allows.
4. **Default SG allows all traffic between resources that share it** — and allows all outbound. Custom SGs deny inbound by default.
5. **Deleting a default egress rule breaks outbound** — a custom SG with the all-outbound rule removed blocks everything until you add specific allows.
6. **SG references are within the same VPC (or peered/attached VPCs with support)** — you can't reference an SG across arbitrary accounts/Regions without peering/TGW support.
7. **NACLs apply to ALL traffic in the subnet**, including cross-AZ and intra-subnet — a too-tight NACL can break internal traffic.
8. **Changes to SGs are near-instant**; NACL changes also apply quickly but existing connections may behave differently (stateless).
9. **The 1024–65535 ephemeral range is required for NAT Gateway** return traffic in NACLs.
10. **Max rules per SG and SGs per ENI interact** — very granular rule sets can hit the combined cap; consolidate with SG references and prefix lists.
11. **Prefix lists count toward rule limits by their max entries**, not current entries — a large managed prefix list can consume many "slots."
12. **You can't attach an SG to a subnet** (that's a NACL) or a NACL to an instance (that's an SG) — they operate at different layers.

---

## Troubleshooting

| Issue                                       | Cause                                              | Fix                                                        |
| ------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------- |
| Connection hangs / times out one-way        | NACL missing ephemeral return range                | Add allow for 1024–65535 in the return direction           |
| Can't reach instance despite open SG        | NACL deny, or wrong subnet NACL                    | Check subnet NACL rules and order                          |
| Can't block a specific attacker IP          | Trying to use an SG deny (doesn't exist)           | Add a NACL deny rule (low number) or use WAF/NFW           |
| Outbound suddenly fails                     | Custom SG egress rule removed                      | Re-add required outbound allows                            |
| Tier-to-tier rule breaks on IP change       | Hardcoded IPs in SG                                | Reference the source SG instead of CIDRs                   |
| "Rules per security group exceeded"         | Too many granular rules                            | Use SG references + managed prefix lists; request increase |
| Intra-subnet traffic blocked                | NACL too restrictive                               | NACLs affect all subnet traffic — widen or use SGs instead |

---

## Best Practices

1. **Use SGs as the primary control**; keep NACLs simple (broad allows + targeted denies).
2. **Reference SGs, not IPs**, for tier-to-tier rules (web→app→db).
3. **Least privilege on ingress**; restrict egress for sensitive tiers rather than leaving all-outbound.
4. **Reserve NACLs for explicit denies** (block known-bad CIDRs) and subnet-wide guardrails.
5. **Always allow ephemeral return ports in NACLs** (1024–65535).
6. **Use managed prefix lists** to keep repeated CIDR sets consistent and within rule limits.
7. **Name and tag SGs by role** (web-sg, app-sg, db-sg) for clarity.
8. **Audit unused SGs** and overly broad `0.0.0.0/0` rules regularly.

---

## Useful Links

- [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Compare SGs and NACLs](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html)
- [Ephemeral ports](https://docs.aws.amazon.com/vpc/latest/userguide/nacl-ephemeral-ports.html)
- [Managed prefix lists](https://docs.aws.amazon.com/vpc/latest/userguide/managed-prefix-lists.html)
- [Security group rules reference](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)

---
