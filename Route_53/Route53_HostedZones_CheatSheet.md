# AWS Route 53 Hosted Zones - Cheat Sheet

## Overview

A hosted zone is a container for DNS records that define how traffic is routed for a domain and its subdomains. Route 53 automatically creates NS and SOA records when you create a hosted zone. Hosted zones come in two types: public (internet-facing) and private (VPC-only).

**Key point:** A hosted zone maps directly to a DNS domain name. You can have multiple hosted zones for the same domain name (split-horizon DNS).

---

## Core Concepts

| Component                       | Description                                                              |
| ------------------------------- | ------------------------------------------------------------------------ |
| **Public Hosted Zone**          | Resolves queries from the internet for a public domain                   |
| **Private Hosted Zone (PHZ)**   | Resolves queries only from associated VPCs                               |
| **NS Records**                  | Name server records — identify the 4 Route 53 name servers for the zone  |
| **SOA Record**                  | Start of Authority — metadata about the zone (serial, refresh, TTL)      |
| **Delegation Set**              | The set of 4 name servers assigned to a hosted zone                      |
| **Reusable Delegation Set**     | A fixed set of 4 NS that can be shared across multiple hosted zones      |
| **Alias Record**                | Route 53-specific record that maps to AWS resources (free, no TTL limit) |
| **Traffic Flow**                | Visual editor for complex routing policies                               |

---

## Public vs. Private Hosted Zones

| Feature                 | Public Hosted Zone                      | Private Hosted Zone                        |
| ----------------------- | --------------------------------------- | ------------------------------------------ |
| Accessible from         | Internet (anyone)                       | Only associated VPCs                       |
| Domain registration     | Required (you must own the domain)      | Not required (any name works)              |
| NS records at registrar | Must point to Route 53 NS              | N/A (no registrar needed)                  |
| VPC association         | Not applicable                          | Required (at least 1 VPC)                  |
| DNSSEC signing          | Supported                               | Not supported                              |
| Health checks           | All routing policies                    | Limited (no endpoint HCs from within VPC)  |
| Routing policies        | All supported                           | Simple, Weighted, Failover, Multivalue, Geolocation, Geoproximity |
| Latency routing         | Supported                               | Not supported                              |
| Query logging           | Supported                               | Supported                                  |
| Split-horizon DNS       | Yes (public half)                       | Yes (private half)                         |

---

## Record Types Supported

| Type    | Description                                         | Example                        |
| ------- | --------------------------------------------------- | ------------------------------ |
| A       | Maps name to IPv4 address                           | `192.0.2.1`                    |
| AAAA    | Maps name to IPv6 address                           | `2001:db8::1`                  |
| CNAME   | Maps name to another domain name                    | `www.example.com`              |
| MX      | Mail exchange servers                               | `10 mail.example.com`          |
| TXT     | Text records (SPF, DKIM, verification)              | `"v=spf1 include:..."`        |
| SRV     | Service locator                                     | `10 5 5269 xmpp.example.com`  |
| NS      | Name server delegation                              | `ns-1.awsdns-01.org`          |
| SOA     | Start of authority                                  | Auto-managed by Route 53      |
| PTR     | Reverse DNS lookup                                  | `host.example.com`            |
| CAA     | Certificate Authority Authorization                 | `0 issue "letsencrypt.org"`   |
| NAPTR   | Name Authority Pointer (SIP/ENUM)                   | Telecom use cases             |
| DS      | Delegation Signer (DNSSEC)                          | Chain of trust to child zone  |

---

## Alias Records

Alias records are a Route 53 extension that lets you route traffic to select AWS resources.

### Alias vs. CNAME

| Feature              | Alias                                   | CNAME                           |
| -------------------- | --------------------------------------- | ------------------------------- |
| Zone apex support    | Yes (`example.com`)                     | No (only subdomains)            |
| Query charges        | Free (for AWS targets)                  | Standard DNS query charges      |
| Health check         | Evaluate Target Health (built-in)       | Requires separate health check  |
| TTL                  | Inherited from target resource          | You set it manually             |
| Non-AWS targets      | Not supported                           | Supported (any domain)          |

### Alias Targets

| AWS Resource                      | Hosted Zone ID Required |
| --------------------------------- | ----------------------- |
| CloudFront distribution           | Z2FDTNDATAQYW2 (fixed) |
| ELB (ALB/NLB/CLB)                | Per-region HZ ID        |
| S3 website endpoint               | Per-region HZ ID        |
| API Gateway (Regional)            | Per-region HZ ID        |
| API Gateway (Edge)                | CloudFront HZ ID        |
| VPC Interface Endpoint            | Per-region HZ ID        |
| Global Accelerator                | Z2BJ6XQ5FK7U4H (fixed) |
| Elastic Beanstalk environment     | Per-region HZ ID        |
| Another Route 53 record (same HZ) | Same hosted zone        |

---

## Private Hosted Zones

### VPC Association Rules

- A PHZ must be associated with at least one VPC
- Can associate VPCs from different regions (cross-region)
- Can associate VPCs from different accounts (cross-account via CLI/API only)
- VPC must have `enableDnsSupport` = true and `enableDnsHostnames` = true

### Resolution Order (VPC DNS)

```
1. Private Hosted Zone (most specific match wins)
2. Resolver Rules (forwarding rules)
3. VPC DNS (auto-generated records: ec2 hostnames, etc.)
4. Public DNS (Route 53 recursive resolver)
```

### Cross-Account PHZ Association

```bash
# Step 1: In PHZ owner account — create authorization
aws route53 create-vpc-association-authorization \
  --hosted-zone-id Z0123456789ABCDEF \
  --vpc VPCRegion=us-east-1,VPCId=vpc-abcdef1234567890

# Step 2: In VPC owner account — associate
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0123456789ABCDEF \
  --vpc VPCRegion=us-east-1,VPCId=vpc-abcdef1234567890

# Step 3: In PHZ owner account — revoke authorization (cleanup)
aws route53 delete-vpc-association-authorization \
  --hosted-zone-id Z0123456789ABCDEF \
  --vpc VPCRegion=us-east-1,VPCId=vpc-abcdef1234567890
```

---

## Reusable Delegation Sets

Allow multiple hosted zones to share the same 4 name servers. Useful for:
- White-label/vanity name servers
- Migrating domains without changing NS records
- Consistent NS across environments

```bash
# Create reusable delegation set
aws route53 create-reusable-delegation-set \
  --caller-reference "my-delegation-set-$(date +%s)"

# Create hosted zone using specific delegation set
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "hz-$(date +%s)" \
  --delegation-set-id N0123456789ABCDEF
```

---

## Split-Horizon DNS

Use the same domain name with both a public and private hosted zone:

```
example.com (Public HZ) → 203.0.113.10 (public-facing servers)
example.com (Private HZ) → 10.0.1.50 (internal servers)
```

- Internet queries resolve to public zone records
- VPC queries resolve to private zone records
- Enables different answers for same domain based on query source

---

## AWS CLI Commands

```bash
# --- HOSTED ZONE MANAGEMENT ---

# Create a public hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "hz-$(date +%s)"

# Create a private hosted zone
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference "hz-$(date +%s)" \
  --vpc VPCRegion=us-east-1,VPCId=vpc-0123456789abcdef0 \
  --hosted-zone-config PrivateZone=true

# List all hosted zones
aws route53 list-hosted-zones

# Get hosted zone details
aws route53 get-hosted-zone --id Z0123456789ABCDEF

# Delete a hosted zone (must delete all non-default records first)
aws route53 delete-hosted-zone --id Z0123456789ABCDEF

# --- RECORD MANAGEMENT ---

# List records in a hosted zone
aws route53 list-resource-record-sets --hosted-zone-id Z0123456789ABCDEF

# Create/Update a record (UPSERT)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z0123456789ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "192.0.2.1"}]
      }
    }]
  }'

# Create an Alias record (ELB example)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z0123456789ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-123456.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Delete a record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z0123456789ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "old.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "192.0.2.99"}]
      }
    }]
  }'

# --- VPC ASSOCIATIONS (Private HZ) ---

# Associate additional VPC with private hosted zone
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0123456789ABCDEF \
  --vpc VPCRegion=us-west-2,VPCId=vpc-abcdef1234567890

# Disassociate VPC from private hosted zone
aws route53 disassociate-vpc-from-hosted-zone \
  --hosted-zone-id Z0123456789ABCDEF \
  --vpc VPCRegion=us-west-2,VPCId=vpc-abcdef1234567890

# --- TRAFFIC FLOW ---

# List traffic policies
aws route53 list-traffic-policies

# Create a traffic policy instance
aws route53 create-traffic-policy-instance \
  --hosted-zone-id Z0123456789ABCDEF \
  --name app.example.com \
  --ttl 60 \
  --traffic-policy-id policy-id \
  --traffic-policy-version 1

# --- TEST DNS ---

# Test DNS answer (simulate query)
aws route53 test-dns-answer \
  --hosted-zone-id Z0123456789ABCDEF \
  --record-name www.example.com \
  --record-type A
```

---

## Pricing

| Component                        | Cost                                   |
| -------------------------------- | -------------------------------------- |
| Hosted zone (first 25)          | $0.50/month per zone                   |
| Hosted zone (over 25)           | $0.10/month per zone                   |
| Standard queries                 | $0.40 per million queries              |
| Latency-based routing queries    | $0.60 per million queries              |
| Geo DNS queries                  | $0.70 per million queries              |
| IP-based routing queries         | $0.80 per million queries              |
| Alias queries to AWS resources   | Free                                   |
| Traffic Flow policy records      | $50.00/month per policy record         |

---

## Quotas

| Resource                              | Default Limit                      |
| ------------------------------------- | ---------------------------------- |
| Hosted zones per account              | 500 (soft limit, can increase)     |
| Records per hosted zone               | 10,000 (soft limit, can increase)  |
| VPC associations per private HZ       | 300                                |
| Reusable delegation sets per account  | 100                                |
| Values per record                     | 400 (for weighted, etc.)           |
| Max TTL                               | 2,147,483,647 seconds              |
| Name servers per hosted zone          | 4 (fixed)                          |

---

## Troubleshooting

| Issue                                    | Cause                                            | Fix                                                          |
| ---------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| Domain not resolving                     | NS at registrar don't match Route 53 HZ          | Update registrar NS to match hosted zone NS records          |
| Private HZ records not resolving in VPC  | VPC DNS settings disabled                        | Enable `enableDnsSupport` and `enableDnsHostnames` on VPC    |
| Wrong zone answering queries             | Multiple HZs for same domain, wrong one matched  | Check most-specific match; verify VPC associations for PHZ   |
| Cannot delete hosted zone                | Non-default records still exist                  | Delete all records except NS and SOA first                   |
| Cross-account PHZ not working            | Authorization not created or VPC not associated  | Create authorization in owner, then associate in VPC account |
| Alias record not resolving               | Target resource doesn't exist or wrong HZ ID     | Verify target resource exists and use correct Alias HZ ID    |
| Propagation delay after changes          | DNS TTL caching                                  | Wait for previous TTL to expire; reduce TTL before changes   |

---

## Best Practices

1. **Lower TTL before making changes** — Reduce TTL to 60s hours before a migration, then restore after
2. **Use Alias records for AWS resources** — Free queries, automatic health checking, zone apex support
3. **Enable Evaluate Target Health on Alias** — Enables automatic failover without separate health checks
4. **Use split-horizon DNS** — Same domain for internal/external with different answers
5. **Use reusable delegation sets** — Consistent NS records simplify migrations and white-labeling
6. **Tag your hosted zones** — Aids cost allocation and organization
7. **Automate record management** — Use IaC or CI/CD pipelines for DNS changes, not manual console edits
8. **Monitor hosted zone limits** — Set CloudWatch alarms as you approach 10,000 records
9. **Use Traffic Flow for complex routing** — Visual editor with versioning is safer than manual record chains
10. **Audit with CloudTrail** — All Route 53 API calls are logged for compliance

---

## Useful Links

- [Hosted Zones Overview](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)
- [Private Hosted Zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html)
- [Alias Records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [Supported DNS Record Types](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html)
- [Reusable Delegation Sets](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-concepts.html#route-53-concepts-reusable-delegation-set)
- [Associating VPCs with Private HZ](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-associate-vpcs.html)
- [Traffic Flow](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/traffic-flow.html)
- [Route 53 Pricing](https://aws.amazon.com/route53/pricing/)

---
