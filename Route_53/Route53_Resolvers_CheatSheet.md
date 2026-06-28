# AWS Route 53 Resolver - Cheat Sheet

## Overview

Route 53 Resolver is the default recursive DNS service for all VPCs. It automatically resolves DNS queries for AWS resources, private hosted zones, and public internet domains. For hybrid environments, you can extend it with **inbound** and **outbound endpoints** to route DNS queries between your VPC and on-premises networks.

---

## How the Default VPC Resolver Works

Every VPC gets a built-in DNS resolver at the VPC's base IP + 2 (e.g., `10.0.0.2` for a `10.0.0.0/16` VPC), also accessible at `169.254.169.253`.

**Default behavior (no endpoints needed):**
1. Queries for VPC-internal names (EC2 hostnames, ELB) → answered locally
2. Queries matching private hosted zones associated with the VPC → answered from the zone
3. All other queries → recursive lookup against public DNS

---

## Architecture Components

| Component             | Direction     | Purpose                                                       |
| --------------------- | ------------- | ------------------------------------------------------------- |
| **Inbound Endpoint**  | On-prem → AWS | Allows on-prem DNS resolvers to forward queries into your VPC |
| **Outbound Endpoint** | AWS → On-prem | Allows VPC Resolver to forward queries to on-prem DNS servers |
| **Forwarding Rules**  | N/A           | Define which domain queries to forward and where              |
| **System Rules**      | N/A           | Auto-defined rules for AWS-internal domains                   |
| **Profiles**          | N/A           | Manage DNS config centrally across VPCs/accounts              |


---

## Inbound Endpoints

**Purpose:** Let DNS resolvers on your on-premises network forward queries to Route 53 Resolver.

**How it works:**
1. You create an inbound endpoint and specify 2+ IP addresses in your VPC subnets
2. Route 53 creates ENIs (Elastic Network Interfaces) for each IP
3. Your on-prem DNS server is configured to forward specific domains to these IPs
4. Queries arrive at the inbound endpoint and are resolved by Route 53 Resolver

**Use cases:**
- On-prem servers need to resolve names in Route 53 private hosted zones
- On-prem apps need to resolve VPC-internal hostnames (EC2, RDS, etc.)
- Centralized DNS resolution for hybrid environments

```
On-Premises DNS Server
        |
        | forwards "aws.internal" queries to Inbound Endpoint IPs
        v
[Inbound Endpoint IPs in VPC]
        |
        v
Route 53 Resolver → Private Hosted Zone / VPC DNS
```

---

## Outbound Endpoints

**Purpose:** Let Route 53 Resolver forward DNS queries from your VPC to DNS servers on your network (or another VPC).

**How it works:**
1. You create an outbound endpoint with 2+ IP addresses in your VPC subnets
2. You create forwarding rules specifying domain → target DNS server IPs
3. You associate rules with VPCs
4. When a query matches a rule, Resolver forwards it via the outbound endpoint

**Use cases:**
- VPC resources need to resolve on-premises Active Directory domains
- VPC resources need to resolve names hosted on corporate DNS
- Forward queries for specific domains to third-party DNS services

```
EC2 instance queries "corp.internal"
        |
        v
Route 53 Resolver (checks rules)
        |
        | matches forwarding rule for "corp.internal"
        v
[Outbound Endpoint]
        |
        v
On-Premises DNS Server (10.1.1.53, 10.1.2.53)
```

---

## Endpoint Configuration Details

| Parameter           | Details                                            |
| ------------------- | -------------------------------------------------- |
| **IP Addresses**    | Minimum 2 per endpoint (for HA across AZs), max 6 |
| **Protocols**       | IPv4, IPv6, or Dual-stack                          |
| **Security Groups** | Applied to endpoint ENIs (control who can query)   |
| **Subnets**         | Recommend different AZs for high availability      |
| **ENI**             | One ENI per IP address, created in your VPC        |


### Capacity & Scaling

| Metric                    | Value                                               |
| ------------------------- | --------------------------------------------------- |
| Queries per second per IP | ~10,000 (max burst, sustained ~1,500-10,000 varies) |
| Minimum IPs per endpoint  | 2                                                   |
| Maximum IPs per endpoint  | 6 (can request increase)                            |
| Scale strategy            | Add more IP addresses to the endpoint               |


> Each IP creates an ENI that can handle DNS queries independently. Spread across AZs for fault tolerance.

---

## Forwarding Rules

### Rule Types

| Type          | Description                                                     |
| ------------- | --------------------------------------------------------------- |
| **Forward**   | Forward queries for this domain to specified target IPs         |
| **System**    | Override forwarding and resolve locally (use Route 53 Resolver) |
| **Recursive** | Default rule for "." (root) — resolves via public DNS           |


### Rule Matching

- **Most specific match wins** (e.g., `acme.example.com` wins over `example.com`)
- Forwarding rules apply to a domain **and all its subdomains** by default
- To exempt a subdomain from forwarding, create a System rule for it

### Auto-Defined System Rules

Route 53 automatically creates system rules for:
- `amazonaws.com` — AWS service endpoints
- `compute.internal` — EC2 internal hostnames
- VPC CIDR reverse DNS (`in-addr.arpa`)
- `localhost`
- Other RFC 6303 reserved zones

> **Tip:** If you create a forwarding rule for "." (root) or "com", also create a System rule for `amazonaws.com` to ensure AWS services still resolve correctly.

---

## Route 53 Profiles

Profiles simplify DNS management across many VPCs and accounts.

**What a Profile can contain:**
- Private hosted zone associations
- Resolver forwarding rules (forward + system)
- DNS Firewall rule groups
- Interface VPC endpoints
- Resolver query logging configurations
- DNSSEC validation settings
- DNS Firewall failure mode settings

**Key characteristics:**
- One profile per VPC
- Regional resource (same region as VPCs)
- Shared across accounts via AWS RAM
- Changes to a profile propagate to all associated VPCs

---

## Resolver Query Logging

Log all DNS queries from your VPC resources.

**What's logged:**
- Query name, type, class
- Response code
- Source instance/ENI
- Timestamp
- VPC and region
- DNS Firewall action (if applicable)
- Firewall domain list ID (for blocked/alerted queries)

**Destinations:**
- CloudWatch Logs
- S3 bucket
- Kinesis Data Firehose

**Key points:**
- Logs only **unique** queries (not cached responses)
- Can be shared across accounts via RAM
- Useful for security auditing and compliance

---

## AWS CLI Commands

```bash
# --- INBOUND ENDPOINT ---

# Create inbound endpoint
aws route53resolver create-resolver-endpoint \
  --creator-request-id "inbound-$(date +%s)" \
  --name "my-inbound-endpoint" \
  --security-group-ids sg-0123456789abcdef0 \
  --direction INBOUND \
  --ip-addresses SubnetId=subnet-aaa,Ip=10.0.1.10 SubnetId=subnet-bbb,Ip=10.0.2.10

# --- OUTBOUND ENDPOINT ---

# Create outbound endpoint
aws route53resolver create-resolver-endpoint \
  --creator-request-id "outbound-$(date +%s)" \
  --name "my-outbound-endpoint" \
  --security-group-ids sg-0123456789abcdef0 \
  --direction OUTBOUND \
  --ip-addresses SubnetId=subnet-aaa,Ip=10.0.1.20 SubnetId=subnet-bbb,Ip=10.0.2.20

# --- FORWARDING RULES ---

# Create forwarding rule
aws route53resolver create-resolver-rule \
  --creator-request-id "rule-$(date +%s)" \
  --name "forward-corp-domain" \
  --rule-type FORWARD \
  --domain-name "corp.internal" \
  --resolver-endpoint-id rslvr-out-abcdef1234567890 \
  --target-ips "Ip=10.1.1.53,Port=53" "Ip=10.1.2.53,Port=53"

# Create system rule (resolve locally, don't forward)
aws route53resolver create-resolver-rule \
  --creator-request-id "system-rule-$(date +%s)" \
  --name "resolve-aws-locally" \
  --rule-type SYSTEM \
  --domain-name "amazonaws.com"

# Associate rule with VPC
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-abcdef1234567890 \
  --vpc-id vpc-0123456789abcdef0 \
  --name "my-rule-assoc"

# --- QUERY & MANAGE ---

# List endpoints
aws route53resolver list-resolver-endpoints

# Get endpoint details
aws route53resolver get-resolver-endpoint \
  --resolver-endpoint-id rslvr-in-abcdef1234567890

# List rules
aws route53resolver list-resolver-rules

# List rule associations
aws route53resolver list-resolver-rule-associations

# Add IP to existing endpoint (scale up)
aws route53resolver associate-resolver-endpoint-ip-address \
  --resolver-endpoint-id rslvr-in-abcdef1234567890 \
  --ip-address SubnetId=subnet-ccc,Ip=10.0.3.10

# Remove IP from endpoint (scale down)
aws route53resolver disassociate-resolver-endpoint-ip-address \
  --resolver-endpoint-id rslvr-in-abcdef1234567890 \
  --ip-address SubnetId=subnet-ccc,Ip=10.0.3.10

# --- QUERY LOGGING ---

# Create query logging config
aws route53resolver create-resolver-query-log-config \
  --name "my-query-log" \
  --destination-arn "arn:aws:logs:us-east-1:123456789012:log-group:/dns/query-logs" \
  --creator-request-id "log-$(date +%s)"

# Associate query logging with VPC
aws route53resolver associate-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-abcdef1234567890 \
  --resource-id vpc-0123456789abcdef0

# --- DNSSEC VALIDATION ---

# Enable DNSSEC validation for a VPC
aws route53resolver update-resolver-dnssec-config \
  --resource-id vpc-0123456789abcdef0 \
  --validation ENABLE

# --- SHARING RULES (RAM) ---

# Share resolver rules with another account
aws ram create-resource-share \
  --name "dns-rules-share" \
  --resource-arns "arn:aws:route53resolver:us-east-1:123456789012:resolver-rule/rslvr-rr-abc" \
  --principals "111122223333"
```

---

## Hybrid DNS Architecture Patterns

### Pattern 1: Central Outbound (Hub-and-Spoke)

```
                    Shared Services VPC (Hub)
                    ┌──────────────────────┐
                    │  Outbound Endpoint   │
                    │  Inbound Endpoint    │
                    │  Forwarding Rules    │
                    └──────────────────────┘
                       ↑         ↓
           ┌───────────┘         └────────────┐
    Spoke VPC A              Spoke VPC B          On-Premises
  (rules via RAM)          (rules via RAM)         DNS Servers
```

- Create endpoints in a central/shared VPC
- Share rules via RAM to spoke accounts
- All DNS forwarding goes through the hub

### Pattern 2: Per-VPC Endpoints (Distributed)

- Endpoints in each VPC that needs hybrid DNS
- More expensive but simpler networking (no transit gateway needed for DNS)
- Each VPC independently forwards

### Pattern 3: Route 53 Profiles (Recommended for Scale)

- Create a Profile with all DNS settings
- Associate Profile with all VPCs across accounts
- Update once, propagates everywhere
- Supports PHZs, rules, DNS Firewall, query logging

---

## Security Groups for Endpoints

### Inbound Endpoint Security Group

| Direction | Protocol | Port | Source                 |
| --------- | -------- | ---- | ---------------------- |
| Inbound   | TCP      | 53   | On-prem DNS server IPs |
| Inbound   | UDP      | 53   | On-prem DNS server IPs |


### Outbound Endpoint Security Group

| Direction | Protocol | Port | Destination            |
| --------- | -------- | ---- | ---------------------- |
| Outbound  | TCP      | 53   | On-prem DNS server IPs |
| Outbound  | UDP      | 53   | On-prem DNS server IPs |


---

## Pricing

| Component                           | Cost                                                       |
| ----------------------------------- | ---------------------------------------------------------- |
| Resolver endpoint (per ENI/IP)      | $0.125/hour (~$90/month per IP)                            |
| DNS queries processed via endpoints | $0.40 per million queries                                  |
| Query logging                       | No charge (you pay for destination: CW Logs, S3, Firehose) |
| Profiles                            | No additional charge                                       |
| Default VPC Resolver                | Free (included with VPC)                                   |


**Minimum cost per endpoint:** 2 IPs × $0.125/hr = $0.25/hr = ~$180/month

> **Cost optimization:** Use a centralized hub VPC with shared rules via RAM instead of deploying endpoints in every VPC.

---

## Quotas

| Resource                           | Default Limit                      |
| ---------------------------------- | ---------------------------------- |
| Resolver endpoints per region      | 4 per direction (inbound/outbound) |
| IP addresses per endpoint          | 6 (min 2)                          |
| Resolver rules per region          | 1,000                              |
| Rules per VPC association          | 500                                |
| Target IPs per forwarding rule     | 6                                  |
| VPCs per resolver rule association | Unlimited                          |
| Query logging configs per region   | 20                                 |
| Profiles per region                | 1 per account (soft limit)         |
| Resources per Profile              | Varies by type                     |


---

## Troubleshooting

| Issue                                   | Cause                                         | Fix                                                       |
| --------------------------------------- | --------------------------------------------- | --------------------------------------------------------- |
| On-prem can't resolve AWS private zones | Inbound endpoint missing or SG blocking       | Create inbound endpoint; allow port 53 from on-prem IPs   |
| VPC can't resolve on-prem domains       | No forwarding rule or outbound endpoint issue | Create rule + outbound endpoint; verify SG & route tables |
| Forwarding rule not taking effect       | Rule not associated with VPC                  | Associate rule with the VPC                               |
| AWS services failing after "." rule     | amazonaws.com being forwarded                 | Add System rule for `amazonaws.com`                       |
| Intermittent resolution failures        | Single-AZ endpoint                            | Use 2+ IPs across different AZs                           |
| Queries timing out via endpoint         | Security group or NACL blocking port 53       | Verify SG allows UDP/TCP 53 in correct direction          |
| Cross-account resolution failing        | Rule not shared or accepted via RAM           | Share via RAM and accept in target account                |


---

## Best Practices

1. **Always use 2+ IPs across different AZs** for endpoint high availability
2. **Centralize endpoints in a shared VPC** and share rules via RAM to reduce costs
3. **Create System rules for amazonaws.com** when using root (".") forwarding rules
4. **Use Route 53 Profiles** for multi-account environments (scales better than per-VPC associations)
5. **Enable query logging** for security auditing and troubleshooting
6. **Restrict security groups** — only allow port 53 from known DNS sources
7. **Monitor endpoint capacity** — scale up (add IPs) before hitting query limits
8. **Use consistent naming** for rules and endpoints to simplify management at scale
9. **Consider DNS64** for IPv6-only subnets needing to resolve IPv4 addresses
10. **Test failover** — verify resolution works if one AZ goes down

---


**R53 Resolver Rule Priority (Highest to Lowest)**

When a DNS query hits the VPC DNS resolver (.2), it's evaluated against rules in this order of precedence:
- Resolver Forwarding Rules (Forward/System rules) — most specific domain match wins
- Private Hosted Zones (autodefined/implicit rules for PHZ associated with the VPC)
- Auto-defined rules (compute.amazonaws.com, compute.internal, reverse DNS for VPC CIDR, RFC 6303 reserved names, peered VPC ranges)
- Default Rule / Internet Resolver (rslvr-autodefined-rr-internet-resolver — the "." dot rule, acts as recursive resolver)

**Key nuance** — the "most specific match" principle applies across all of these:
- If multiple rules overlap, the most specific domain always wins regardless of rule type.
- For example: If you have a Forward Rule for example.com. but a PHZ for sub.example.com. is associated with the VPC, the PHZ's implicit rule wins for sub.example.com. because it's more specific.
- A customer-created Forward or System rule can override both auto-defined rules and PHZ implicit rules by creating a rule for the exact same domain name.

**Exceptions (from the wiki):**
- Even with a CatchAll "." Forward Rule, the Resolver will NOT forward queries for auto-defined rules (internal IP ranges not accessible from outside AWS) — unless the customer explicitly:
  - Sets enableDnsHostnames for the VPC to false, or
  - Creates explicit rules for those auto-defined domain names

**So in practice: explicit customer rules > PHZ implicit rules > auto-defined rules > default internet resolver, with most-specific-domain-match being the tie-breaker at each level except private HZ with the auto-defined rules.**

---


## Useful Links

- [What is Route 53 VPC Resolver?](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)
- [Forwarding Inbound Queries (On-prem → VPC)](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-forward-network-to-vpc.html)
- [Forwarding Outbound Queries (VPC → On-prem)](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html)
- [Managing Forwarding Rules](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-rules-managing.html)
- [Auto-defined System Rules](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-forward-vpc-to-network-autodefined-rules.html)
- [Sharing Rules via RAM](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-rules-managing-sharing.html)
- [Route 53 Profiles](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/profiles.html)
- [Resolver Query Logging](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs.html)
- [Endpoint Scaling Best Practices](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-resolver-endpoint-scaling.html)
- [HA with Resolver Endpoints (Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-achieve-dns-high-availability-with-route-53-resolver-endpoints/)
- [Migrating to Route 53 Profiles (Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-your-multi-account-dns-environment-to-amazon-route-53-profiles/)

---

*Sources: AWS official documentation and AWS blogs. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
