# AWS Route 53 Routing Policies - Cheat Sheet

## Overview

Route 53 routing policies determine how DNS queries are answered. They let you control traffic distribution across multiple resources based on criteria like weight, latency, geography, or health status.

---

## All Routing Policies at a Glance

| Policy                | Use Case                                    | Health Check Support       | Private HZ |
| --------------------- | ------------------------------------------- | -------------------------- | ---------- |
| **Simple**            | Single resource or multiple values (random) | No (per record)            | Yes        |
| **Weighted**          | Distribute traffic by proportion            | Yes                        | Yes        |
| **Latency**           | Route to lowest-latency region              | Yes                        | No         |
| **Failover**          | Active-passive DR                           | Yes (required for primary) | Yes        |
| **Geolocation**       | Route by user's geographic location         | Yes                        | Yes        |
| **Geoproximity**      | Route by distance to resource + bias        | Yes                        | Yes        |
| **Multivalue Answer** | Return multiple healthy IPs (up to 8)       | Yes                        | Yes        |
| **IP-based**          | Route by source IP/CIDR range               | Yes                        | No         |


---

## 1. Simple Routing

**What it does:** Routes traffic to a single resource. You can specify multiple values in one record; Route 53 returns all values in random order.

**Key Points:**
- One record per name/type combination
- Can include multiple IPs in a single record (returned in random order)
- Cannot attach health checks to individual values within the record
- If multiple values are specified and one is unhealthy, Route 53 still returns all values

**When to use:**
- Single server / single endpoint
- No need for health-check-aware routing

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "example.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [
        {"Value": "192.0.2.1"},
        {"Value": "192.0.2.2"}
      ]
    }
  }]
}'
```

---

## 2. Weighted Routing

**What it does:** Distributes traffic across multiple resources in proportions you define.

**Key Points:**
- Weight range: 0-255
- Traffic proportion = record weight / sum of all weights
- Weight 0 = no traffic (unless all records are 0, then equal distribution)
- Each record needs a unique **Set Identifier**
- Supports health checks (unhealthy records are excluded from calculation)

**Formula:** `% traffic = weight / (sum of all weights)`

**When to use:**
- Canary deployments (send 10% to new version)
- A/B testing
- Load distribution across regions
- Gradual migration between environments

```bash
# Record A: 70% traffic
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "server-1",
      "Weight": 70,
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.1"}]
    }
  }]
}'

# Record B: 30% traffic
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "server-2",
      "Weight": 30,
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.2"}]
    }
  }]
}'
```

> **Gotcha:** DNS caching means the distribution is approximate, not exact. Use low TTLs for more responsive weight changes.

---

## 3. Latency-Based Routing

**What it does:** Routes users to the AWS region with the lowest network latency from their location.

**Key Points:**
- Route 53 maintains a latency database between user locations and AWS regions
- You specify which region each resource is in
- Each record needs a unique Set Identifier + Region
- Only considers AWS regions (not arbitrary locations)
- Latency changes over time; routing decisions can shift

**When to use:**
- Multi-region applications
- Minimizing response time for global users
- Active-active multi-region architectures

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "us-east-1",
      "Region": "us-east-1",
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.1"}]
    }
  }]
}'
```

> **Note:** Latency-based routing is NOT available for private hosted zones.

---

## 4. Failover Routing

**What it does:** Active-passive failover. Routes to primary unless it's unhealthy, then routes to secondary.

**Key Points:**
- Exactly 2 records: **Primary** and **Secondary**
- Primary MUST have a health check
- Secondary health check is optional (if omitted, always considered healthy)
- When primary recovers, traffic fails back to primary
- Use alias records to point to ELBs, CloudFront, etc.

**When to use:**
- Disaster recovery (DR)
- Active-passive architectures
- Maintenance windows with a static "sorry" page

```bash
# Primary record
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "primary",
      "Failover": "PRIMARY",
      "TTL": 60,
      "HealthCheckId": "hc-1234",
      "ResourceRecords": [{"Value": "192.0.2.1"}]
    }
  }]
}'

# Secondary record
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "secondary",
      "Failover": "SECONDARY",
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.2"}]
    }
  }]
}'
```

---

## 5. Geolocation Routing

**What it does:** Routes traffic based on the geographic location of the user (where the DNS query originates).

**Key Points:**
- Locations can be specified at: **Continent**, **Country**, or **State** (US only)
- Priority: State > Country > Continent > Default
- You SHOULD create a **default record** for unmatched locations
- Without a default record, unmatched locations get "no answer" (NXDOMAIN-like)
- Some IP addresses can't be mapped to a location
- Works for both public and private hosted zones (private uses VPC region)

**When to use:**
- Content localization (language-specific servers)
- Compliance / data residency (keep EU data in EU)
- Restricting content distribution by geography
- Predictable load balancing by region

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "europe",
      "GeoLocation": {"ContinentCode": "EU"},
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.10"}]
    }
  }]
}'
```

**Continent Codes:** AF (Africa), AN (Antarctica), AS (Asia), EU (Europe), NA (North America), OC (Oceania), SA (South America)

---

## 6. Geoproximity Routing

**What it does:** Routes traffic based on geographic distance between users and resources, with an adjustable **bias** to expand or shrink routing regions.

**Key Points:**
- Routes to the **closest** resource by geographic distance
- **Bias** range: -99 to +99
  - Positive bias: expands the area (attracts more traffic)
  - Negative bias: shrinks the area (pushes traffic away)
  - 0 = no bias (pure distance)
- Resources can be in **AWS regions** or **custom coordinates** (latitude/longitude)
- Requires **Traffic Flow** (visual policy editor) when using coordinates
- Can be used without Traffic Flow for AWS-region-based resources

**When to use:**
- Shift traffic between regions without changing infrastructure
- Route to the nearest data center including non-AWS locations
- Fine-tune geographic traffic distribution

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "us-east",
      "GeoProximityLocation": {
        "AWSRegion": "us-east-1",
        "Bias": 25
      },
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.1"}]
    }
  }]
}'
```

### Geolocation vs Geoproximity

| Feature               | Geolocation                                    | Geoproximity                    |
| --------------------- | ---------------------------------------------- | ------------------------------- |
| Based on              | Political boundaries (continent/country/state) | Physical distance               |
| Adjustable?           | No                                             | Yes (bias -99 to +99)           |
| Non-AWS locations     | No                                             | Yes (lat/long coordinates)      |
| "Default" record      | Yes (catch-all)                                | Routes to nearest automatically |
| Traffic Flow required | No                                             | Only for custom coordinates     |


---

## 7. Multivalue Answer Routing

**What it does:** Returns up to 8 healthy IP addresses in response to each DNS query.

**Key Points:**
- Returns up to **8 healthy records** per query
- Each record can have its own health check
- If all records are unhealthy, returns up to 8 unhealthy records (fail-open)
- Different resolvers may get different random subsets
- NOT a load balancer replacement (client picks one IP from the list)
- Alias records NOT supported with multivalue answer

**When to use:**
- DNS-level availability improvement
- Client-side failover (try next IP if first fails)
- Simple traffic distribution without weighted logic

**Simple vs Multivalue:**

| Feature              | Simple (multiple values) | Multivalue Answer              |
| -------------------- | ------------------------ | ------------------------------ |
| Health checks        | Not supported per value  | Supported per record           |
| Unhealthy removal    | No                       | Yes (excluded from response)   |
| Max records returned | All values               | Up to 8                        |


```bash
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "server-1",
      "MultiValueAnswer": true,
      "TTL": 60,
      "HealthCheckId": "hc-1234",
      "ResourceRecords": [{"Value": "192.0.2.1"}]
    }
  }]
}'
```

---

## 8. IP-Based Routing

**What it does:** Routes traffic based on the source IP (CIDR range) of the DNS query.

**Key Points:**
- You create a **CIDR collection** with named locations, each containing CIDR blocks
- Records are associated with a CIDR location
- Longest prefix match is used when CIDRs overlap
- Useful for ISP-specific routing, enterprise networks with known IP ranges
- You need a **default** record for IPs not in any CIDR collection

**When to use:**
- Route enterprise customers from known IP ranges to specific endpoints
- ISP-specific optimizations (route ISP-A traffic to endpoint closest to their network)
- Migrate from geolocation to more precise IP-based routing
- Internal service routing based on source network

```bash
# Create a CIDR collection
aws route53 create-cidr-collection --name "my-cidrs" --caller-reference "unique-ref-1"

# Add CIDR blocks to a location within the collection
aws route53 change-cidr-collection \
  --id collection-id \
  --changes '[{
    "Action": "PUT",
    "LocationName": "office-nyc",
    "CidrList": ["203.0.113.0/24", "198.51.100.0/24"]
  }]'

# Create an IP-based record
aws route53 change-resource-record-sets --hosted-zone-id Z1234 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "nyc-office",
      "CidrRoutingConfig": {
        "CollectionId": "collection-id",
        "LocationName": "office-nyc"
      },
      "TTL": 60,
      "ResourceRecords": [{"Value": "192.0.2.50"}]
    }
  }]
}'
```

---

## Combining Routing Policies (Complex Configurations)

You can build **routing trees** by combining policies using alias records:

```
                    example.com
                         |
              Latency (pick region)
               /                  \
         us-east-1            eu-west-1
             |                    |
      Weighted (blue/green)   Failover
        /         \            /      \
   v1 (90%)   v2 (10%)   Primary  Secondary
```

**Rules for combining:**
- Use **alias records** to chain routing policies together
- Parent record points to child record set (same hosted zone)
- Each level applies its own routing logic
- Health checks can exist at every level (use `Evaluate Target Health` for alias records)

---

## Traffic Flow (Visual Policy Editor)

Traffic Flow is a graphical interface for building complex routing configurations.

**Key features:**
- Visual drag-and-drop editor for routing trees
- Versioned policies (create new versions without affecting live traffic)
- Policy records: apply a traffic policy to a specific domain name
- Required for geoproximity with custom lat/long coordinates
- Supports all routing types in combination

**Pricing:** $50/month per policy record (after first policy record which is also $50)

```bash
# List traffic policies
aws route53 list-traffic-policies

# Get traffic policy details
aws route53 get-traffic-policy --id policy-id --version 1

# Create a policy record (apply policy to a domain)
aws route53 create-traffic-policy-instance \
  --hosted-zone-id Z1234 \
  --name "app.example.com" \
  --ttl 60 \
  --traffic-policy-id policy-id \
  --traffic-policy-version 1
```

---

## Alias Records

Alias records are a Route 53-specific extension that route traffic to AWS resources.

**Supported targets:**
- CloudFront distributions
- Elastic Load Balancers (ALB, NLB, CLB)
- API Gateway
- S3 website endpoints
- VPC interface endpoints
- Another Route 53 record in the same hosted zone
- Elastic Beanstalk environments
- AWS Global Accelerator

**Key advantages over CNAME:**
- Work at zone apex (e.g., `example.com` not just `www.example.com`)
- No charge for alias queries to AWS resources
- `Evaluate Target Health` option (inherits target's health automatically)
- Return the resource's A/AAAA record directly (no extra DNS hop)

---

## Health Check Integration with Routing

| Scenario                       | Behavior                                 |
| ------------------------------ | ---------------------------------------- |
| Record healthy                 | Included in responses                    |
| Record unhealthy               | Excluded from responses                  |
| All records unhealthy          | Route 53 returns all records (fail-open) |
| No health check on record      | Record always considered healthy         |
| Alias + Evaluate Target Health | Uses target resource's health            |
| Failover primary unhealthy     | Returns secondary                        |


---

## Key Quotas

| Resource                           | Limit                             |
| ---------------------------------- | --------------------------------- |
| Records per hosted zone            | 10,000 (soft limit, can increase) |
| Weighted records per name/type     | 100                               |
| Latency records per name/type      | One per region                    |
| Geolocation records per name/type  | One per location                  |
| CIDR collections per account       | 5                                 |
| CIDR blocks per CIDR collection    | 1,000                             |
| Locations per CIDR collection      | 100                               |
| Traffic policies per account       | 50                                |
| Traffic policy records per account | 5                                 |


---

## Common Mistakes

1. **Missing default record in geolocation** → Users from unmapped locations get no answer
2. **Not associating health checks** → Unhealthy endpoints still receive traffic
3. **High TTL with failover** → Failover takes effect only after cached records expire
4. **Weight 0 confusion** → Weight 0 stops traffic UNLESS all records are 0
5. **Latency routing in private HZ** → Not supported; use geolocation or geoproximity instead
6. **Alias without Evaluate Target Health** → Alias record won't react to target failures
7. **Missing Set Identifier** → Required for all policies except Simple

---

## Quick Decision Guide

| Situation                                   | Which Policy to use |
| ------------------------------------------- | ------------------- |
| Need DR / active-passive?                   | Failover            |
| Need to split traffic by %?                 | Weighted            |
| Need lowest latency for global users?       | Latency-based       |
| Need routing by country/region?             | Geolocation         |
| Need routing by distance + adjustable bias? | Geoproximity        |
| Need routing by source IP/CIDR?             | IP-based            |
| Need multiple healthy IPs returned?         | Multivalue Answer   |
| Just one resource, nothing fancy?           | Simple              |



---

## Useful Links

- [Choosing a Routing Policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Weighted Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html)
- [Latency Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
- [Geolocation Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)
- [Geoproximity Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geoproximity.html)
- [IP-based Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-ipbased.html)
- [Multivalue Answer Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-multivalue.html)
- [Configuring DNS Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-configuring.html)
- [Traffic Flow Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/traffic-policies.html)
- [IP-based Routing Blog Post](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-ip-based-routing-for-amazon-route-53)

---

*Sources: AWS official documentation and AWS blogs. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
