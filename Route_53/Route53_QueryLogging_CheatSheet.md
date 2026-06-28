# AWS Route 53 Resolver Query Logging - Cheat Sheet

## Overview

Route 53 Resolver Query Logging captures information about DNS queries that originate from resources within your VPCs (including EC2 instances, Lambda functions, containers, etc.). It logs all DNS queries handled by the VPC Resolver — whether they go to public DNS, private hosted zones, forwarding rules, or are filtered by DNS Firewall.

**Key point:** Query logging captures queries from VPC resources through the VPC Resolver (AmazonProvidedDNS at 169.254.169.253 / VPC+2 address). It does NOT log queries to public hosted zones from the internet — use CloudTrail or CloudWatch for that.

---

## Core Concepts

| Component                     | Description                                                            |
| ----------------------------- | ---------------------------------------------------------------------- |
| **Query Log Config**          | A named configuration that defines where to send logs                  |
| **VPC Association**           | Links a query log config to a VPC to start logging                     |
| **Log Destination**           | Where logs are sent: CloudWatch Logs, S3, or Kinesis Data Firehose     |
| **Resolver Query Log**        | The actual log entries containing DNS query details                     |
| **Sharing (RAM)**             | Share query log configs across accounts via AWS Resource Access Manager |

---

## How Query Logging Works

```
VPC Resource makes DNS query (e.g., EC2 instance)
        |
        v
VPC Resolver (AmazonProvidedDNS / .2 address)
        |
        v
Query is logged (if VPC has associated query log config)
        |
        v
Log sent to destination:
  - CloudWatch Logs (near real-time, ~seconds)
  - S3 (batched, every few minutes)
  - Kinesis Data Firehose (near real-time, streaming)
```

### What Gets Logged

- All queries from VPC resources to the VPC Resolver
- Queries forwarded via Resolver rules (outbound)
- Queries received via inbound endpoints
- Queries filtered by DNS Firewall (with firewall-specific fields)
- Queries to private hosted zones
- Queries to public DNS (recursive resolution)

### What Does NOT Get Logged

- Queries from the internet to public hosted zones (use CloudTrail)
- DNS queries that bypass the VPC Resolver (e.g., direct to 8.8.8.8)
- Queries between Route 53 Resolver and upstream authoritative servers

---

## Log Destinations

| Destination            | Latency         | Use Case                                      | Format     |
| ---------------------- | --------------- | --------------------------------------------- | ---------- |
| **CloudWatch Logs**    | Near real-time  | Real-time monitoring, alerting, quick analysis | JSON       |
| **S3**                 | Minutes (batch) | Long-term storage, compliance, cost-effective  | JSON (gzip)|
| **Kinesis Firehose**   | Near real-time  | Streaming to Splunk, Elasticsearch, Redshift   | JSON       |

### Destination ARN Formats

```
CloudWatch Logs: arn:aws:logs:REGION:ACCOUNT:log-group:LOG_GROUP_NAME
S3:              arn:aws:s3:::BUCKET_NAME
Firehose:        arn:aws:firehose:REGION:ACCOUNT:deliverystream/STREAM_NAME
```

---

## Log Record Fields

### Standard Fields

| Field                  | Description                                              | Example                          |
| ---------------------- | -------------------------------------------------------- | -------------------------------- |
| `version`              | Log format version                                       | `1.100000`                       |
| `account_id`          | AWS account ID                                           | `123456789012`                   |
| `region`              | AWS region                                               | `us-east-1`                      |
| `vpc_id`             | VPC where the query originated                           | `vpc-0123456789abcdef0`          |
| `query_timestamp`    | When the query was made (ISO 8601)                       | `2024-01-15T10:30:00Z`           |
| `query_name`         | Domain name queried                                      | `www.example.com.`               |
| `query_type`         | DNS record type                                          | `A`, `AAAA`, `CNAME`, `MX`      |
| `query_class`        | DNS query class                                          | `IN`                             |
| `rcode`              | Response code                                            | `NOERROR`, `NXDOMAIN`, `SERVFAIL`|
| `answers`            | DNS response records                                     | Array of answer objects          |
| `srcaddr`            | Source IP of the querying resource                        | `10.0.1.50`                      |
| `srcport`            | Source port                                              | `52345`                          |
| `transport`          | Protocol used                                            | `UDP` or `TCP`                   |
| `srcids`             | Source resource identifiers                               | Instance ID, ENI ID              |

### DNS Firewall Fields (when applicable)

| Field                      | Description                                    | Example                    |
| -------------------------- | ---------------------------------------------- | -------------------------- |
| `firewall_rule_action`     | Action taken (ALLOW, BLOCK, ALERT)             | `BLOCK`                    |
| `firewall_rule_group_id`   | Rule group that matched                        | `rslvr-frg-abcdef123`     |
| `firewall_domain_list_id`  | Domain list that matched                       | `rslvr-fdl-abcdef123`     |

### Resolver Endpoint Fields

| Field                      | Description                                    | Example                    |
| -------------------------- | ---------------------------------------------- | -------------------------- |
| `resolver_endpoint`        | Resolver endpoint ID (if query used one)       | `rslvr-in-abcdef123`      |
| `edns_client_subnet`       | EDNS client subnet if present                  | `203.0.113.0/24`           |

---

## Sample Log Entry

```json
{
  "version": "1.100000",
  "account_id": "123456789012",
  "region": "us-east-1",
  "vpc_id": "vpc-0123456789abcdef0",
  "query_timestamp": "2024-01-15T10:30:00Z",
  "query_name": "malware.badsite.com.",
  "query_type": "A",
  "query_class": "IN",
  "rcode": "NOERROR",
  "answers": [
    {
      "Rdata": "0.0.0.0",
      "Type": "A",
      "Class": "IN"
    }
  ],
  "srcaddr": "10.0.1.50",
  "srcport": "52345",
  "transport": "UDP",
  "srcids": {
    "instance": "i-0abcdef1234567890"
  },
  "firewall_rule_action": "BLOCK",
  "firewall_rule_group_id": "rslvr-frg-abcdef1234567890",
  "firewall_domain_list_id": "rslvr-fdl-abcdef1234567890"
}
```

---

## AWS CLI Commands

```bash
# --- QUERY LOG CONFIG MANAGEMENT ---

# Create query log config (CloudWatch Logs destination)
aws route53resolver create-resolver-query-log-config \
  --name "vpc-dns-logs" \
  --destination-arn "arn:aws:logs:us-east-1:123456789012:log-group:/dns/query-logs" \
  --creator-request-id "log-$(date +%s)"

# Create query log config (S3 destination)
aws route53resolver create-resolver-query-log-config \
  --name "vpc-dns-logs-s3" \
  --destination-arn "arn:aws:s3:::my-dns-logs-bucket" \
  --creator-request-id "log-$(date +%s)"

# Create query log config (Kinesis Firehose destination)
aws route53resolver create-resolver-query-log-config \
  --name "vpc-dns-logs-firehose" \
  --destination-arn "arn:aws:firehose:us-east-1:123456789012:deliverystream/dns-log-stream" \
  --creator-request-id "log-$(date +%s)"

# List query log configs
aws route53resolver list-resolver-query-log-configs

# Get query log config details
aws route53resolver get-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-abcdef1234567890

# Delete query log config (must disassociate all VPCs first)
aws route53resolver delete-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-abcdef1234567890

# --- VPC ASSOCIATIONS ---

# Associate query log config with a VPC
aws route53resolver associate-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-abcdef1234567890 \
  --resource-id vpc-0123456789abcdef0

# List VPC associations
aws route53resolver list-resolver-query-log-config-associations

# Disassociate query log config from a VPC
aws route53resolver disassociate-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-abcdef1234567890 \
  --resource-id vpc-0123456789abcdef0

# --- SHARING (RAM) ---

# Share query log config with another account
aws ram create-resource-share \
  --name "dns-query-log-share" \
  --resource-arns "arn:aws:route53resolver:us-east-1:123456789012:resolver-query-log-config/rqlc-abcdef1234567890" \
  --principals "111122223333"

# --- QUERYING LOGS (CloudWatch Logs Insights) ---

# Example: Find all blocked queries in the last hour
# Run in CloudWatch Logs Insights console:
# fields @timestamp, query_name, srcaddr, firewall_rule_action
# | filter firewall_rule_action = "BLOCK"
# | sort @timestamp desc
# | limit 100

# Example: Top queried domains
# fields query_name
# | stats count(*) as query_count by query_name
# | sort query_count desc
# | limit 20

# Example: Find queries from a specific instance
# fields @timestamp, query_name, query_type, rcode
# | filter srcids.instance = "i-0abcdef1234567890"
# | sort @timestamp desc
```

---

## CloudWatch Logs Insights Queries

### Security Analysis

```
# Find NXDOMAIN responses (potential DGA activity)
fields @timestamp, query_name, srcaddr
| filter rcode = "NXDOMAIN"
| stats count(*) as nxdomain_count by srcaddr
| sort nxdomain_count desc
| limit 10

# Find DNS queries to unusual ports (potential tunneling)
fields @timestamp, query_name, srcaddr, query_type
| filter query_type = "TXT" or query_type = "NULL"
| sort @timestamp desc
| limit 100

# DNS Firewall blocked queries summary
fields query_name, srcaddr, firewall_rule_action
| filter firewall_rule_action = "BLOCK"
| stats count(*) as block_count by query_name
| sort block_count desc
| limit 20
```

### Operational Analysis

```
# Top talkers (source IPs making most queries)
fields srcaddr
| stats count(*) as query_count by srcaddr
| sort query_count desc
| limit 10

# Query volume over time
fields @timestamp
| stats count(*) as queries by bin(5m)
| sort @timestamp asc

# Failed queries (SERVFAIL)
fields @timestamp, query_name, srcaddr, rcode
| filter rcode = "SERVFAIL"
| sort @timestamp desc
| limit 50
```

---

## Pricing

| Component                        | Cost                                               |
| -------------------------------- | -------------------------------------------------- |
| Query logging itself             | Free (no charge from Route 53)                     |
| CloudWatch Logs ingestion        | $0.50 per GB ingested                              |
| CloudWatch Logs storage          | $0.03 per GB/month                                 |
| S3 storage                       | Standard S3 pricing (~$0.023 per GB/month)         |
| Kinesis Data Firehose            | $0.029 per GB (first 500 TB/month)                 |

> Route 53 does not charge for query logging. You only pay for the log destination service.

---

## Quotas

| Resource                                    | Default Limit                |
| ------------------------------------------- | ---------------------------- |
| Query log configs per region per account    | 20                           |
| VPC associations per query log config       | 100                          |
| Query log configs per VPC                   | 1 (only one config per VPC)  |

---

## Required IAM Permissions

### For Creating Query Log Configs

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53resolver:CreateResolverQueryLogConfig",
        "route53resolver:AssociateResolverQueryLogConfig",
        "route53resolver:ListResolverQueryLogConfigs",
        "route53resolver:GetResolverQueryLogConfig"
      ],
      "Resource": "*"
    }
  ]
}
```

### Resource Policies (Destination Permissions)

Route 53 Resolver needs permission to write to your destination:

**CloudWatch Logs** — Route 53 creates a resource policy automatically when you create the log config.

**S3** — Add this bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-dns-logs-bucket/AWSLogs/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::my-dns-logs-bucket"
    }
  ]
}
```

**Kinesis Data Firehose** — Route 53 needs an IAM role with `firehose:PutRecord` and `firehose:PutRecordBatch` permissions.

---

## S3 Log Path Structure

When logging to S3, files are organized as:

```
s3://bucket-name/AWSLogs/ACCOUNT_ID/vpcdnsquerylogs/VPC_ID/YYYY/MM/DD/
```

Files are gzip-compressed JSON, delivered every few minutes.

---

## Troubleshooting

| Issue                                    | Cause                                          | Fix                                                        |
| ---------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| No logs appearing                        | Query log config not associated with VPC       | Associate the config with the target VPC                   |
| Missing firewall fields in logs          | Query didn't hit any DNS Firewall rule         | Firewall fields only appear when a rule matches            |
| Logs delayed                             | S3 delivers in batches (minutes); CW is faster | Switch to CloudWatch Logs for near real-time               |
| Permission denied creating config        | Missing IAM permissions                        | Add route53resolver and destination permissions            |
| S3 logs not delivered                    | Missing bucket policy for delivery service     | Add the S3 resource policy for `delivery.logs.amazonaws.com`|
| Cannot associate VPC                     | VPC already has a query log config associated  | Disassociate existing config first (1 per VPC limit)       |
| Cannot delete query log config           | Still associated with VPCs                     | Disassociate all VPCs first, then delete                   |
| Log volume too high / expensive          | Noisy VPC with many queries                    | Filter in CloudWatch, use S3 for cost savings, or archive  |

---

## Best Practices

1. **Use CloudWatch Logs for security monitoring** — Near real-time visibility for threat detection
2. **Use S3 for compliance/archival** — Cost-effective long-term storage with lifecycle policies
3. **Use Kinesis Firehose for SIEM integration** — Stream to Splunk, Elasticsearch, or Datadog
4. **Set up CloudWatch Alarms** — Alert on high NXDOMAIN rates, blocked queries, or SERVFAIL spikes
5. **Enable for all VPCs** — Consistent logging ensures no blind spots
6. **Use CloudWatch Logs Insights** — Powerful query language for ad-hoc analysis
7. **Combine with DNS Firewall** — Query logs show firewall actions for auditing and tuning
8. **Set S3 lifecycle policies** — Transition to Glacier after 30-90 days, delete after retention period
9. **Use RAM for multi-account** — Share a centralized query log config across accounts
10. **Monitor log volume** — High DNS query volume can generate significant log costs
11. **Correlate with VPC Flow Logs** — Combine DNS logs with network flow logs for full visibility
12. **Tag your configs** — Aids cost allocation and management

---

## Integration with AWS Services

| Service                    | Integration                                                     |
| -------------------------- | --------------------------------------------------------------- |
| **CloudWatch Logs**        | Real-time log storage, Insights queries, metric filters         |
| **CloudWatch Alarms**      | Alert on suspicious DNS patterns                                |
| **S3**                     | Long-term archival with lifecycle management                    |
| **Kinesis Data Firehose**  | Stream to third-party tools (Splunk, Elasticsearch)             |
| **Athena**                 | Query S3-stored logs with SQL                                   |
| **AWS RAM**                | Share query log configs across accounts                         |
| **Security Hub**           | Correlate DNS findings with other security data                 |
| **Detective**              | Investigate DNS-related security findings                        |
| **GuardDuty**              | DNS-based threat detection complements query logging            |
| **Lambda**                 | Process logs in real-time via CloudWatch Logs subscription       |

---

## Architecture Patterns

### Pattern 1: Centralized Security Monitoring

```
All VPCs → Query Log Config → CloudWatch Logs
                                     |
                                     v
                            CloudWatch Metric Filters
                                     |
                                     v
                            SNS Alarm → Security Team
```

### Pattern 2: Multi-Account Compliance

```
Central Account creates Query Log Config
        |
        v
Share via RAM to all member accounts
        |
        v
Each account associates with their VPCs
        |
        v
All logs → Central S3 bucket → Athena for analysis
```

### Pattern 3: SIEM Integration

```
VPCs → Query Log Config → Kinesis Data Firehose
                                    |
                                    v
                          Transformation (Lambda)
                                    |
                                    v
                          Splunk / Elasticsearch / Datadog
```

---

## Useful Links

- [Query Logging Overview](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs.html)
- [Configuring Query Logging](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs-choosing-target-resource.html)
- [Query Log Format](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs-format.html)
- [Sharing Query Log Configs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs-sharing.html)
- [CloudWatch Logs Insights Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [DNS Firewall Logging](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/firewall-resolver-query-logs.html)

---
