# AWS Route 53 Profiles - Cheat Sheet

## Overview

Route 53 Profiles allow you to manage DNS configurations centrally and apply them consistently across multiple VPCs and accounts. A Profile is a container that groups together DNS-related resources (resolver rules, firewall rule groups, private hosted zone associations) so they can be applied as a single unit.

**Key point:** Profiles simplify multi-VPC and multi-account DNS management by eliminating the need to individually associate each resource with every VPC.

---

## Core Concepts

| Component                    | Description                                                              |
| ---------------------------- | ------------------------------------------------------------------------ |
| **Profile**                  | A named container grouping DNS configuration resources together          |
| **Resource Association**     | Linking a DNS resource (rule, PHZ, firewall group) to a Profile          |
| **VPC Association**          | Attaching a Profile to a VPC to apply all its grouped resources          |
| **Sharing (RAM)**            | Sharing Profiles across accounts via AWS Resource Access Manager          |
| **Owner Account**            | The account that creates and manages the Profile                         |
| **Consumer Account**         | An account that accepts a shared Profile and associates it with VPCs     |

---

## How Profiles Work

```
Owner Account creates Profile
        |
        v
Add resources to Profile:
  - Private Hosted Zone associations
  - Resolver Rules (forwarding rules)
  - DNS Firewall Rule Groups
        |
        v
Share Profile via AWS RAM (optional)
        |
        v
Associate Profile with VPCs (same or different accounts)
        |
        v
All resources in the Profile are automatically applied to associated VPCs
```

### Key Behavior

1. One Profile per VPC (a VPC can only have one Profile associated at a time)
2. Changes to the Profile are automatically reflected in all associated VPCs
3. Resources within a Profile maintain their individual priority/ordering
4. Removing a resource from a Profile removes it from all associated VPCs

---

## Resources You Can Add to a Profile

| Resource Type                | What It Does When Associated                          |
| ---------------------------- | ----------------------------------------------------- |
| **Private Hosted Zone (PHZ)** | VPC can resolve records in that private hosted zone  |
| **Resolver Forwarding Rule** | VPC forwards matching queries to specified targets    |
| **DNS Firewall Rule Group**  | VPC enforces the firewall rules in that group         |
| **Resolver Query Log Configurations**  | Query logging for attached VPC will be stored         |

---

## Profile vs. Direct Association

| Feature                      | Direct Association               | Profile-Based                          |
| ---------------------------- | -------------------------------- | -------------------------------------- |
| Management                   | Per-VPC, per-resource            | Centralized in one Profile             |
| Multi-VPC updates            | Update each VPC individually     | Update Profile once, applies to all    |
| Cross-account sharing        | RAM per resource                 | RAM the entire Profile                 |
| VPC limit                    | N/A                              | 1 Profile per VPC                      |
| Use case                     | Simple, few VPCs                 | Large-scale, multi-account             |

---

## Use Cases

### 1. Multi-Account DNS Standardization

Central networking account creates a Profile with:
- Forwarding rules for on-prem domains
- DNS Firewall rules for security
- Shared private hosted zones

Share via RAM → All spoke accounts associate with their VPCs.

### 2. Environment Separation

```
Profile: "production-dns"
  - PHZ: prod.internal.company.com
  - Firewall: strict security rules
  - Forwarding: prod on-prem DNS

Profile: "development-dns"
  - PHZ: dev.internal.company.com
  - Firewall: relaxed rules (alert only)
  - Forwarding: dev on-prem DNS
```

### 3. Consistent Security Baseline

```
Profile: "security-baseline"
  - DNS Firewall: Block malware domains
  - DNS Firewall: Block DNS tunneling
  - DNS Firewall: Alert on newly observed domains
```

Apply to all VPCs across all accounts for consistent protection.

---

## AWS CLI Commands

```bash
# --- PROFILE MANAGEMENT ---

# Create a Profile
aws route53profiles create-profile \
  --name "central-dns-profile"

# List Profiles
aws route53profiles list-profiles

# Get Profile details
aws route53profiles get-profile \
  --profile-id rp-abcdef1234567890

# Delete a Profile (must have no associations)
aws route53profiles delete-profile \
  --profile-id rp-abcdef1234567890

# --- RESOURCE ASSOCIATIONS ---

# Associate a Private Hosted Zone with a Profile
aws route53profiles associate-resource-to-profile \
  --profile-id rp-abcdef1234567890 \
  --name "prod-phz" \
  --resource-arn "arn:aws:route53:::hostedzone/Z0123456789ABCDEF"

# Associate a Resolver Rule with a Profile
aws route53profiles associate-resource-to-profile \
  --profile-id rp-abcdef1234567890 \
  --name "onprem-forwarding" \
  --resource-arn "arn:aws:route53resolver:us-east-1:123456789012:resolver-rule/rslvr-rr-abcdef"

# Associate a DNS Firewall Rule Group with a Profile
aws route53profiles associate-resource-to-profile \
  --profile-id rp-abcdef1234567890 \
  --name "security-firewall" \
  --resource-arn "arn:aws:route53resolver:us-east-1:123456789012:firewall-rule-group/rslvr-frg-abcdef" \
  --resource-properties '{"priority": 100}'

# List resources in a Profile
aws route53profiles list-profile-resource-associations \
  --profile-id rp-abcdef1234567890

# Disassociate a resource from a Profile
aws route53profiles disassociate-resource-from-profile \
  --profile-id rp-abcdef1234567890 \
  --resource-arn "arn:aws:route53:::hostedzone/Z0123456789ABCDEF"

# --- VPC ASSOCIATIONS ---

# Associate Profile with a VPC
aws route53profiles associate-profile \
  --profile-id rp-abcdef1234567890 \
  --name "prod-vpc-assoc" \
  --resource-id vpc-0123456789abcdef0

# List VPC associations for a Profile
aws route53profiles list-profile-associations

# Disassociate Profile from a VPC
aws route53profiles disassociate-profile \
  --profile-association-id rpa-abcdef1234567890

# --- SHARING (RAM) ---

# Share Profile with another account
aws ram create-resource-share \
  --name "dns-profile-share" \
  --resource-arns "arn:aws:route53profiles:us-east-1:123456789012:profile/rp-abcdef1234567890" \
  --principals "111122223333"

# Share Profile with entire Organization
aws ram create-resource-share \
  --name "org-dns-profile" \
  --resource-arns "arn:aws:route53profiles:us-east-1:123456789012:profile/rp-abcdef1234567890" \
  --principals "arn:aws:organizations::123456789012:organization/o-abcdef"
```

---

## Pricing

Route 53 Profiles are billed based on the number of VPC associations. The **Profile owner** is responsible for the bill, including VPC associations made by consumer accounts.

| Component                              | Cost                          |
| -------------------------------------- | ----------------------------- |
| First 100 VPC associations (per hour)  | $0.75/hr (flat rate)          |
| Each VPC association beyond 100        | $0.0014/hr per association    |
| Profile creation                       | No additional charge          |
| Resource sharing (RAM)                 | No additional charge          |

### Pricing Example

An account creates a Route 53 Profile in us-east-1 associated with **200 VPCs** in a 30-day month:

```
Total cost = [$0.75 (first 100 VPCs) + (100 additional VPCs × $0.0014)] × (24 hrs × 30 days)
           = [$0.75 + $0.14] × 720
           = $0.89 × 720
           = $640.80/month
```

> You also continue to pay for the underlying resources (resolver endpoints, firewall queries, hosted zones, etc.)

---

## Quotas

| Resource                              | Default Limit                |
| ------------------------------------- | ---------------------------- |
| Profiles per region per account       | 1 (soft limit, can increase) |
| Resource associations per Profile     | Varies by resource type      |
| VPC associations per Profile          | Unlimited                    |
| Profiles per VPC                      | 1 (hard limit)               |

---

## Troubleshooting

| Issue                                        | Cause                                           | Fix                                                      |
| -------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| Cannot associate Profile with VPC            | VPC already has a Profile associated            | Disassociate existing Profile first                      |
| Resource not taking effect in VPC            | Profile not associated with that VPC            | Associate the Profile with the target VPC                |
| Shared Profile not visible in consumer acct  | RAM share not accepted                          | Accept the RAM resource share in the consumer account    |
| Cannot delete Profile                        | Profile still has VPC associations              | Disassociate all VPCs first, then delete                 |
| DNS Firewall priority conflict               | Multiple firewall groups with same priority     | Set unique priorities via resource-properties             |
| PHZ records not resolving                    | Profile association pending or VPC DNS disabled | Check association status; ensure enableDnsSupport is on  |

---

## Best Practices

1. **Use Profiles for multi-account environments** — Centralizes DNS management and ensures consistency
2. **One Profile per environment tier** — Separate profiles for prod, staging, dev with appropriate resources
3. **Share via RAM at the Organization level** — Reduces per-account sharing overhead
4. **Monitor Profile associations** — Track which VPCs have which Profiles in your CMDB
5. **Use naming conventions** — Consistent names for Profiles and resource associations aid auditing
6. **Plan for the 1-per-VPC limit** — Since a VPC can only have one Profile, include all needed resources in that one Profile
7. **Test in non-production first** — Associating a Profile applies all its resources immediately
8. **Use with AWS Firewall Manager** — Combine Profiles with FMS for Organization-wide DNS security
9. **Document ownership** — Clearly define which team owns the Profile vs. who consumes it
10. **Version control your configurations** — Use IaC (CloudFormation/Terraform) to manage Profile resources

---

## Integration with AWS Services

| Service                    | Integration                                                     |
| -------------------------- | --------------------------------------------------------------- |
| **AWS RAM**                | Share Profiles across accounts and Organizations                |
| **AWS Organizations**      | Apply Profiles across all member accounts                       |
| **AWS Firewall Manager**   | Enforce DNS Firewall policies via Profiles                      |
| **CloudFormation**         | Create and manage Profiles as infrastructure-as-code            |
| **Terraform**              | `aws_route53profiles_profile` resource for IaC management       |
| **CloudTrail**             | Audit all Profile API actions                                   |

---

## Useful Links

- [Route 53 Profiles Overview](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/profiles.html)
- [Creating and Managing Profiles](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/profiles-creating-managing.html)
- [Sharing Profiles with RAM](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/profiles-sharing.html)
- [Associating Profiles with VPCs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/profiles-associating-vpc.html)
- [Route 53 Profiles API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/API_Operations_Amazon_Route_53_Profiles.html)

---
