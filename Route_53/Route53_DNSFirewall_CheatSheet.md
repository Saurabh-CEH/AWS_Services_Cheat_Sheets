# AWS Route 53 Resolver DNS Firewall - Cheat Sheet

## Overview

Route 53 Resolver DNS Firewall filters and regulates outbound DNS traffic from your VPCs. It allows you to block DNS queries to known malicious domains, enforce domain allowlists, and protect against data exfiltration via DNS tunneling.

**Key point:** DNS Firewall is a feature of Route 53 VPC Resolver. No additional Resolver setup is required to use it.

---

## Core Concepts

| Component              | Description                                                             |
| ---------------------- | ----------------------------------------------------------------------- |
| **Domain List**        | A collection of domain names to match against (custom or AWS-managed)   |
| **Rule**               | Matches a domain list + specifies an action (ALLOW, BLOCK, ALERT)       |
| **Rule Group**         | A named collection of rules with priorities                             |
| **VPC Association**    | Links a rule group to a VPC, enabling filtering                         |
| **Priority**           | Numeric value determining rule evaluation order (lower = evaluated first)|
| **Managed Domain Lists** | AWS-curated lists of known malicious domains (free)                   |


---

## How DNS Firewall Works

```
VPC Resource makes DNS query
        |
        v
Route 53 Resolver
        |
        v
DNS Firewall evaluates rules (lowest priority first)
        |
        ├── Match found → Execute action (ALLOW / BLOCK / ALERT)
        |
        └── No match → Continue to next rule group / allow by default
```

### Evaluation Order

1. Rule groups are evaluated in **priority order** (lowest number first)
2. Within a rule group, rules are evaluated in **priority order** (lowest first)
3. First matching rule wins — no further evaluation
4. If no rule matches across all rule groups → query is **allowed** (default open)

---

## Actions

| Action    | Behavior                                                |
| --------- | ------------------------------------------------------- |
| **ALLOW** | Permit the query. Stop evaluating further rules.        |
| **BLOCK** | Deny the query. Return a configurable response.         |
| **ALERT** | Permit the query but log it. Continue evaluating rules. |


### BLOCK Response Types

| Response Type | Description                                                          |
| ------------- | -------------------------------------------------------------------- |
| **NODATA**    | Returns empty response (no error, no records)                        |
| **NXDOMAIN**  | Returns "domain does not exist"                                      |
| **OVERRIDE**  | Returns a custom DNS response (e.g., redirect to a walled garden IP) |


---

## Domain Lists

### Custom Domain Lists

- You create and manage these lists
- Add specific domains you want to allow, block, or alert on
- Support wildcard matching: `*.example.com` matches all subdomains
- Plain `example.com` matches `example.com` AND all its subdomains

### AWS Managed Domain Lists (Free)

AWS maintains lists of domains associated with malicious activity:

| List                            | Description                                  |
| ------------------------------- | -------------------------------------------- |
| **AmazonGuardDutyThreatList**   | Domains identified by GuardDuty as threats   |
| **AmazonRegisteredDomainsList** | Used with allowlists for AWS service domains |
| Botnet C&C domains              | Command-and-control infrastructure           |
| Malware domains                 | Known malware distribution sites             |
| Newly observed domains          | Recently registered domains (higher risk)    |


> Managed lists are free to use. You reference them in rules just like custom lists.

---

## DNS Firewall Advanced (Threat Protection)

Beyond static domain lists, DNS Firewall Advanced provides intelligent threat detection:

### Threat Types Detected

| Threat                                | Description                                                 |
| ------------------------------------- | ----------------------------------------------------------- |
| **DNS Tunneling**                     | Data exfiltration by encoding data in DNS queries/responses |
| **DGA (Domain Generation Algorithm)** | Malware-generated random domain names for C&C              |


### How Advanced Detection Works

- Inspects DNS payload characteristics: query string patterns, length, type, frequency, timing
- Identifies suspicious signatures in real-time
- Does NOT rely on static domain lists — uses behavioral analysis
- Available as a rule type within rule groups

### Advanced Rule Actions

Same as standard rules: ALLOW, BLOCK, ALERT

---

## VPC Failure Mode Configuration

Controls what happens when DNS Firewall is unavailable:

| Mode                    | Behavior                                           |
| ----------------------- | -------------------------------------------------- |
| **Fail Open** (default) | If firewall fails, queries pass through unfiltered |
| **Fail Closed**         | If firewall fails, all queries are blocked         |


> Choose based on your risk tolerance: fail-open preserves availability, fail-closed preserves security.

---

## Architecture Patterns

### Pattern 1: Blocklist (Block Known Bad)

```
Rule Group Priority 100:
  Rule 1 (Priority 1): Block → Malware domain list
  Rule 2 (Priority 2): Block → Custom bad domains
  Rule 3 (Priority 3): Alert → Newly observed domains
```

All other queries are allowed by default.

### Pattern 2: Allowlist (Allow Known Good, Block Everything Else)

```
Rule Group Priority 100:
  Rule 1 (Priority 1): Allow → AWS service domains
  Rule 2 (Priority 2): Allow → Your approved domains
  Rule 3 (Priority 9999): Block → All domains (wildcard "*")
```

**Important:** Place the catch-all block rule at the highest priority number (last evaluated).

### Pattern 3: Combined (Recommended)

```
Rule Group Priority 100 (Threat Protection):
  Rule 1: Block → Malware managed list
  Rule 2: Block → Botnet managed list
  Rule 3: Alert → DGA detection (Advanced)
  Rule 4: Alert → DNS tunneling detection (Advanced)

Rule Group Priority 200 (Custom Policy):
  Rule 1: Allow → Internal domains
  Rule 2: Block → Social media domains (during work hours)
  Rule 3: Alert → Newly observed domains
```

---

## AWS CLI Commands

```bash
# --- DOMAIN LISTS ---

# Create a custom domain list
aws route53resolver create-firewall-domain-list \
  --creator-request-id "list-$(date +%s)" \
  --name "blocked-domains"

# Add domains to the list
aws route53resolver update-firewall-domains \
  --firewall-domain-list-id rslvr-fdl-abcdef1234567890 \
  --operation ADD \
  --domains "malware.example.com" "*.phishing.net" "badsite.org"

# Remove domains from the list
aws route53resolver update-firewall-domains \
  --firewall-domain-list-id rslvr-fdl-abcdef1234567890 \
  --operation REMOVE \
  --domains "badsite.org"

# Import domains from file (S3)
aws route53resolver import-firewall-domains \
  --firewall-domain-list-id rslvr-fdl-abcdef1234567890 \
  --operation REPLACE \
  --domain-file-url "s3://my-bucket/blocked-domains.txt"

# List managed domain lists
aws route53resolver list-firewall-domain-lists

# --- RULE GROUPS ---

# Create a rule group
aws route53resolver create-firewall-rule-group \
  --creator-request-id "rg-$(date +%s)" \
  --name "security-rules"

# Create a BLOCK rule
aws route53resolver create-firewall-rule \
  --creator-request-id "rule-$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abcdef1234567890 \
  --firewall-domain-list-id rslvr-fdl-abcdef1234567890 \
  --priority 100 \
  --action BLOCK \
  --block-response NXDOMAIN \
  --name "block-malware"

# Create an ALLOW rule
aws route53resolver create-firewall-rule \
  --creator-request-id "rule-$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abcdef1234567890 \
  --firewall-domain-list-id rslvr-fdl-allowed1234567890 \
  --priority 50 \
  --action ALLOW \
  --name "allow-approved-domains"

# Create an ALERT rule
aws route53resolver create-firewall-rule \
  --creator-request-id "rule-$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abcdef1234567890 \
  --firewall-domain-list-id rslvr-fdl-suspicious1234 \
  --priority 200 \
  --action ALERT \
  --name "alert-suspicious"

# Create BLOCK rule with custom override response
aws route53resolver create-firewall-rule \
  --creator-request-id "rule-$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abcdef1234567890 \
  --firewall-domain-list-id rslvr-fdl-abcdef1234567890 \
  --priority 150 \
  --action BLOCK \
  --block-response OVERRIDE \
  --block-override-domain-name "blocked.company.com" \
  --block-override-dns-type CNAME \
  --block-override-ttl 60 \
  --name "block-redirect-to-walled-garden"

# --- VPC ASSOCIATION ---

# Associate rule group with VPC
aws route53resolver associate-firewall-rule-group \
  --creator-request-id "assoc-$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abcdef1234567890 \
  --vpc-id vpc-0123456789abcdef0 \
  --priority 100 \
  --name "security-rules-for-prod-vpc"

# List associations
aws route53resolver list-firewall-rule-group-associations

# Disassociate
aws route53resolver disassociate-firewall-rule-group \
  --firewall-rule-group-association-id rslvr-frgassoc-abcdef

# --- VPC FAILURE MODE ---

# Update VPC DNS Firewall config (fail open/closed)
aws route53resolver update-firewall-config \
  --resource-id vpc-0123456789abcdef0 \
  --firewall-fail-open ENABLED

# Get firewall config for VPC
aws route53resolver get-firewall-config \
  --resource-id vpc-0123456789abcdef0

# --- QUERY & LIST ---

# List rule groups
aws route53resolver list-firewall-rule-groups

# List rules in a group
aws route53resolver list-firewall-rules \
  --firewall-rule-group-id rslvr-frg-abcdef1234567890

# List domains in a list
aws route53resolver list-firewall-domains \
  --firewall-domain-list-id rslvr-fdl-abcdef1234567890

# --- SHARING (RAM) ---

# Share rule group with another account
aws ram create-resource-share \
  --name "dns-firewall-share" \
  --resource-arns "arn:aws:route53resolver:us-east-1:123456789012:firewall-rule-group/rslvr-frg-abc" \
  --principals "111122223333"
```

---

## CloudWatch Metrics

|             Metric             |               Description               |
--------------------------------|-----------------------------------------
| `FirewallRuleGroupQueryVolume` | Total queries evaluated by a rule group |
|     `FirewallRuleHitCount`     | Number of times a specific rule matched |

---

## Logging

DNS Firewall actions are logged via **Resolver Query Logging**.

### Log Fields for Firewall Events


|           Field           |        Description        |
----------------------------|--------------------------
|  `firewall_rule_action`   |  ALLOW, BLOCK, or ALERT   |
| `firewall_rule_group_id`  | Which rule group matched  |
| `firewall_domain_list_id` | Which domain list was hit |


### Setting Up Logging

```bash
# Create query log config (logs go to CloudWatch Logs)
aws route53resolver create-resolver-query-log-config \
  --name "dns-firewall-logs" \
  --destination-arn "arn:aws:logs:us-east-1:123456789012:log-group:/dns/firewall" \
  --creator-request-id "log-$(date +%s)"

# Associate with VPC
aws route53resolver associate-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-abcdef1234567890 \
  --resource-id vpc-0123456789abcdef0
```

---

## Integration with AWS Services

|          Service           |                                Integration                                |
--------------------------------------|-----------------------------------------------------------------
|  **AWS Firewall Manager**  | Centrally deploy DNS Firewall policies across accounts in an Organization |
|        **AWS RAM**         |                     Share rule groups across accounts                     |
|   **Route 53 Profiles**    |   Include DNS Firewall rule groups in profile for multi-VPC management    |
|       **CloudWatch**       |                      Metrics and alarms on rule hits                      |
| **Resolver Query Logging** |                   Full audit trail of firewall actions                    |
|       **GuardDuty**        |               Managed threat lists from GuardDuty findings                |

---

## Pricing


|              Component               |                Cost                 |
---------------------------------------|------------------------------------
| Domain list queries (first 1B/month) |      $0.60 per million queries      |
| Domain list queries (over 1B/month)  |      $0.40 per million queries      |
|        DNS Firewall Advanced         |      Additional cost per query      |
|       AWS Managed domain lists       |                Free                 |
|     Firewall Manager policy fee      | $100/month per region (if using FM) |


> Billing applies to all DNS queries from VPCs with associated rule groups AND queries traversing inbound Resolver endpoints.

---

## Quotas


|                     Resource                     | Default Limit |
--------------------------------------------------|---------------
|       Domain lists per account per region        |     1,000     |
|             Domains per domain list              |    100,000    |
|        Rule groups per account per region        |     1,000     |
|               Rules per rule group               |      100      |
|         Rule group associations per VPC          |       5       |
| Total rules across all associated groups per VPC |      500      |
|    Firewall domain list file size (S3 import)    |     35 MB     |


---

## Domain Matching Behavior


|     Pattern     |                                     Matches                                      |
-----------------|----------------------------------------------------------------------------------
|  `example.com`  |                `example.com` AND `*.example.com` (all subdomains)                |
| `*.example.com` | Only subdomains: `sub.example.com`, `a.b.example.com` (NOT `example.com` itself) |
|       `*`       |                         Everything (catch-all wildcard)                          |


> This is different from standard DNS wildcards. A bare domain in the list matches itself AND all subdomains.

---

## Troubleshooting


|               Issue               |                               Cause                               |                                 Fix                                  |
-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------+
|    Legitimate domains blocked     |            Overly broad domain list or priority issue             | Check rule priority order; add ALLOW rule with lower priority number |
|         Block not working         | Rule group not associated with VPC, or higher-priority ALLOW rule |              Verify association; check rule priorities               |
|       Inconsistent blocking       |                   Domain list propagation delay                   |         Wait a few minutes; changes propagate asynchronously         |
| Query log missing firewall fields |               Query logging not associated with VPC               |               Associate query log config with the VPC                |
|        All queries failing        |             Fail-closed mode + firewall service issue             |          Switch to fail-open or investigate firewall health          |
|     Advanced rules not firing     |             Advanced not enabled or wrong threat type             |             Verify rule configuration for DGA/tunneling              |


---

## Best Practices

1. **Start with ALERT before BLOCK** — Monitor matches before enforcing blocks to avoid disrupting legitimate traffic
2. **Use managed domain lists** — Free, maintained by AWS, covers known threats
3. **Layer your defenses** — Combine managed lists (threats) + custom lists (policy) + Advanced (behavioral)
4. **Set appropriate priorities** — ALLOW rules should have lower priority numbers than BLOCK rules in allowlist patterns
5. **Enable query logging** — Essential for auditing, incident response, and rule tuning
6. **Use Firewall Manager** for multi-account deployments — Ensures consistent policy across an Organization
7. **Use Route 53 Profiles** — Associate DNS Firewall rule groups with Profiles for easy multi-VPC management
8. **Test with ALERT first** — Validate your rules won't break applications before switching to BLOCK
9. **Review blocked domains regularly** — False positives happen; tune your lists based on alerts
10. **Consider fail-open for availability-critical workloads** — Fail-closed can cause outages if the firewall service has issues
11. **Protect against DNS tunneling** — Enable DNS Firewall Advanced for behavioral detection
12. **Use mutation protection** — Lock down rule group associations to prevent accidental removal

---

## Security Patterns

### Data Exfiltration Prevention

```
Rule Group:
  1. Allow → AWS service domains (amazonaws.com, etc.)
  2. Allow → Your approved SaaS domains
  3. Block → DNS tunneling (Advanced)
  4. Block → DGA detection (Advanced)
  5. Alert → All other queries (or block for strict environments)
```

### Compliance / Regulatory

```
Rule Group:
  1. Block → Gambling domains
  2. Block → Social media (during business hours - use Lambda to rotate lists)
  3. Block → Known malware (managed list)
  4. Alert → Newly observed domains
```

---

## Useful Links

- [DNS Firewall Overview](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall.html)
- [How DNS Firewall Works](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-overview.html)
- [Getting Started with DNS Firewall](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-getting-started.html)
- [Rule Actions](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-rule-actions.html)
- [Managed Domain Lists](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-managed-domain-lists.html)
- [Custom Domain Lists](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-user-managed-domain-lists.html)
- [DNS Firewall Advanced](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/firewall-advanced.html)
- [VPC Failure Configuration](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-vpc-configuration.html)
- [Sharing Rule Groups (RAM)](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-rule-group-sharing.html)
- [Configuring Logging](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/firewall-resolver-query-logs-configuring.html)
- [Protect Against Advanced DNS Threats (Blog)](https://aws.amazon.com/blogs/security/protect-against-advanced-dns-threats-with-amazon-route-53-resolver-dns-firewall/)
- [Centralizing Domain List Management (Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/centralizing-domain-list-management-for-aws-network-firewall-and-route-53-resolver-dns-firewall/)

---
