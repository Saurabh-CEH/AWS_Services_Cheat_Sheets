# AWS Route 53 Health Checks - Cheat Sheet

## Overview

Route 53 health checks monitor the health and performance of your endpoints (web servers, applications, other resources). They integrate with DNS routing to automatically remove unhealthy endpoints from DNS responses.

**SLA:** 100% availability SLA for Route 53 DNS; health checks are part of the global service.

---

## Types of Health Checks

| Type                    | Description                                                                  |
| ----------------------- | ---------------------------------------------------------------------------- |
| **Endpoint**            | Monitors a specific IP/domain via HTTP, HTTPS, or TCP                        |
| **Calculated**          | Monitors the status of other health checks (parent/child)                    |
| **CloudWatch Alarm**    | Monitors the state of a CloudWatch alarm (OK = healthy, ALARM = unhealthy)   |
| **Recovery Controller** | Used with Application Recovery Controller (ARC) for routing control failover |


---

## Health Check Protocols & Behavior

### Endpoint Monitoring

| Protocol            | Behavior                                                                          |
| ------------------- | --------------------------------------------------------------------------------- |
| **HTTP**            | TCP connect + send HTTP GET request; expects 2xx or 3xx within 2 seconds          |
| **HTTPS**           | TCP connect + TLS handshake + send HTTPS GET; expects 2xx or 3xx within 2 seconds |
| **TCP**             | Only checks TCP connection can be established                                     |
| **HTTP_STR_MATCH**  | HTTP check + searches first 5,120 bytes of response body for a specified string   |
| **HTTPS_STR_MATCH** | HTTPS check + searches first 5,120 bytes of response body for a specified string  |


### Timeouts

| Protocol             | TCP Connection Timeout | Response Timeout                                                  |
| -------------------- | ---------------------- | ----------------------------------------------------------------- |
| HTTP/HTTPS           | 4 seconds              | 2 seconds (after connection)                                      |
| TCP                  | 10 seconds             | N/A                                                               |
| HTTP/HTTPS STR_MATCH | 4 seconds              | 2 seconds (after connection) + receive response body in 2 seconds |


---

## Request Interval & Failure Threshold

| Setting               | Standard                               | Fast                            |
| --------------------- | -------------------------------------- | ------------------------------- |
| **Request Interval**  | Every 30 seconds                       | Every 10 seconds                |
| **Failure Threshold** | 1-10 (default: 3) consecutive failures | Same                            |
| **Cost**              | Base price                             | Additional cost for fast checks |


> Health checkers from different data centers don't coordinate. You may see several requests per second followed by pauses.

---

## How Health Status Is Determined

1. Each health checker independently evaluates the endpoint based on **response time** and **failure threshold**.
2. Route 53 aggregates results from all health checkers globally.
3. **Healthy:** More than 18% of health checkers report the endpoint as healthy.
4. **Unhealthy:** 18% or fewer of health checkers report healthy.

### CloudWatch Metric
- `HealthCheckStatus`: 1 = healthy, 0 = unhealthy

---

## Calculated Health Checks

Combine multiple child health checks using logic:

| Logic                  | Description                                             |
| ---------------------- | ------------------------------------------------------- |
| **AND**                | Healthy only if ALL child checks pass                   |
| **OR**                 | Healthy if at least one child check passes              |
| **NOT**                | Healthy if a specific check fails                       |
| **Threshold (x of n)** | Healthy if at least x out of n child checks are healthy |


- Max 256 child health checks per calculated health check.

---

## Health Check Regions

Route 53 health checkers are distributed globally across multiple AWS regions. You can:
- Choose which regions to use (minimum 3 regions required)
- Use all available regions (recommended for critical resources)
- Health checkers run on 16 redundant availability zones

### Key Regions for Health Checking
- us-east-1 (N. Virginia)
- us-west-1 (N. California)
- us-west-2 (Oregon)
- eu-west-1 (Ireland)
- ap-southeast-1 (Singapore)
- ap-southeast-2 (Sydney)
- ap-northeast-1 (Tokyo)
- sa-east-1 (Sao Paulo)

---

## IP Ranges & Firewall Configuration

Health checkers come from specific IP ranges. To allow health check traffic:

```bash
# Get health checker IP ranges
aws route53 get-checker-ip-ranges

# Recommended: Download full AWS IP ranges JSON
curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | \
  jq '.prefixes[] | select(.service=="ROUTE53_HEALTHCHECKS")'
```

**Firewall Rule:** Allow inbound traffic from all Route 53 health check IP ranges on the port your health check uses.

> The service name in ip-ranges.json is `ROUTE53_HEALTHCHECKS`

---

## Configuration Options

### Endpoint Health Check Parameters

| Parameter               | Description                                                                 |
| ----------------------- | --------------------------------------------------------------------------- |
| **IP Address**          | IPv4 or IPv6 of the endpoint                                                |
| **Domain Name**         | FQDN (if no IP specified, resolves via DNS)                                 |
| **Port**                | 1-65535 (default: 80 for HTTP, 443 for HTTPS)                               |
| **Resource Path**       | URL path for HTTP/HTTPS checks (e.g., `/health`)                            |
| **Search String**       | String to find in response body (max 255 chars, first 5,120 bytes searched) |
| **Request Interval**    | 10 or 30 seconds                                                            |
| **Failure Threshold**   | 1-10 consecutive failures before marking unhealthy                          |
| **Enable SNI**          | Send TLS SNI for HTTPS checks (recommended: true)                           |
| **Regions**             | Choose specific regions or use all                                          |
| **Invert Health Check** | Flip healthy/unhealthy status                                               |
| **Latency Graphs**      | Enable latency measurement in CloudWatch                                    |


### Advanced Options

| Parameter                         | Description                                                 |
| --------------------------------- | ----------------------------------------------------------- |
| **Health Threshold (Calculated)** | Number of child checks that must be healthy                 |
| **Insufficient Data Handling**    | What status to assign when CloudWatch has insufficient data |

---

## DNS Failover Patterns

### Active-Passive (Failover Policy)
- Primary record with health check
- Secondary record (no health check needed, always treated as healthy)
- Traffic goes to secondary only when primary health check fails

### Active-Active (Weighted/Latency/Multivalue)
- Multiple records, each with a health check
- Route 53 only returns records with healthy endpoints
- If all fail, Route 53 returns all records (fail-open behavior)

### Complex Configurations
- Combine alias records (weighted alias, failover alias) with non-alias records
- Use latency alias to pick region, weighted within region for individual servers
- Associate health checks at each level of the routing tree

---

## Pricing (approximate, verify current pricing)

| Item                                                                           | Cost                         |
| ------------------------------------------------------------------------------ | ---------------------------- |
| AWS endpoint health checks                                                     | $0.50/month per check        |
| Non-AWS endpoint health checks                                                 | $0.75/month per check        |
| Optional features (HTTPS, string matching, fast interval, latency measurement) | Additional $1.00-$2.00/month |
| First 50 AWS endpoint health checks                                            | Free tier eligible           |


> String matching, HTTPS, fast interval (10s), and latency measurement each add cost.

---

## CloudWatch Integration

### Health Check Metrics (Namespace: `AWS/Route53`)

| Metric                         | Description                                 |
| ------------------------------ | ------------------------------------------- |
| `HealthCheckStatus`            | 1 = healthy, 0 = unhealthy                  |
| `HealthCheckPercentageHealthy` | % of health checkers reporting healthy      |
| `ConnectionTime`               | Time to establish TCP connection (ms)       |
| `SSLHandshakeTime`             | Time to complete SSL/TLS handshake (ms)     |
| `TimeToFirstByte`              | Time to receive first byte of response (ms) |


### Setting Up Alarms
- Health check metrics are published to **us-east-1** region in CloudWatch
- Can trigger SNS notifications on status changes
- Configure alarm actions for auto-remediation

---

## AWS CLI Commands

```bash
# Create a health check
aws route53 create-health-check \
  --caller-reference "unique-string-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "example.com",
    "Port": 443,
    "RequestInterval": 30,
    "FailureThreshold": 3,
    "EnableSNI": true
  }'

# List all health checks
aws route53 list-health-checks

# Get health check status (for development/debug only, not production monitoring)
aws route53 get-health-check-status --health-check-id <id>

# Get health check details
aws route53 get-health-check --health-check-id <id>

# Update a health check
aws route53 update-health-check \
  --health-check-id <id> \
  --failure-threshold 5

# Delete a health check
aws route53 delete-health-check --health-check-id <id>

# Get checker IP ranges
aws route53 get-checker-ip-ranges

# Get health check count
aws route53 get-health-check-count
```

---

## Important Constraints

| Constraint                            | Value                                  |
| ------------------------------------- | -------------------------------------- |
| Health checks per account             | 200 (soft limit, can request increase) |
| Child health checks per calculated HC | 256 max                                |
| String search max length              | 255 characters                         |
| String search max body scanned        | First 5,120 bytes                      |
| Minimum regions to select             | 3                                      |
| Health checkers are **outside VPC**   | Must use public IP for endpoint checks |
| Health check API region               | Always `us-east-1` (global service)    |


---

## Troubleshooting

### Common Causes of Unhealthy Status

1. **Firewall/Security Group blocking health checker IPs** - Allow Route 53 health check IP ranges
2. **Endpoint responding too slowly** - HTTP/HTTPS must respond within 2s after TCP connection
3. **Wrong HTTP status code** - Must return 2xx or 3xx (redirects count as healthy for HTTP, but 302 breaks string matching)
4. **String not found** - Ensure string appears in first 5,120 bytes of response body
5. **Private endpoint (no public IP)** - Health checkers are outside VPC; endpoint must be publicly reachable
6. **DNS resolution failure** - If using domain name, it must resolve to a routable IP
7. **TCP connection timeout** - Connection must complete within 4s (HTTP/HTTPS) or 10s (TCP)
8. **SNI mismatch** - Enable SNI if your HTTPS endpoint requires it

### Debugging Steps

```bash
# 1. Check health check status from all checkers
aws route53 get-health-check-status --health-check-id <id>

# 2. Test connectivity from your machine
curl -o /dev/null -s -w "TCP: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://your-endpoint/health

# 3. Check if response contains your search string (first 5120 bytes)
curl -s https://your-endpoint/health | head -c 5120 | grep "your-string"

# 4. Verify security group allows Route 53 IPs
aws route53 get-checker-ip-ranges --output json

# 5. Check CloudWatch for health check metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Route53 \
  --metric-name HealthCheckStatus \
  --dimensions Name=HealthCheckId,Value=<id> \
  --start-time $(date -v-1H -u +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average \
  --region us-east-1
```

---

## Best Practices

1. **Use multiple health check regions** for critical resources to reduce impact of regional outages
2. **Set appropriate failure thresholds** - too low = flapping, too high = slow detection
3. **Use string matching** for deeper health validation (not just TCP/HTTP status)
4. **Monitor the right layer** - health check should test actual application functionality, not just web server
5. **Consider DNS TTL** - failover won't take effect until cached DNS records expire (recommended TTL: 60s for failover records)
6. **Use calculated health checks** to create composite health signals for complex applications
7. **Set up CloudWatch alarms** on health check metrics for proactive notification
8. **Regularly review orphaned health checks** - unused checks still cost money and generate traffic to endpoints
9. **Use fast interval (10s)** for critical production endpoints to reduce detection time
10. **Test failover regularly** - simulate failures to validate your DR process works end-to-end

---

## Key Differences: Health Checks vs. ELB Health Checks

| Feature     | Route 53 Health Checks         | ELB Health Checks                 |
| ----------- | ------------------------------ | --------------------------------- |
| Scope       | Global (checks from worldwide) | Regional (checks from within VPC) |
| Purpose     | DNS-level failover             | Target group traffic routing      |
| Protocol    | HTTP, HTTPS, TCP               | HTTP, HTTPS, TCP, gRPC            |
| Location    | External (public internet)     | Internal (VPC)                    |
| Integration | DNS records, failover routing  | Load balancer target groups       |


---

## Useful Links

- [Route 53 Health Checks Overview](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html)
- [Types of Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-types.html)
- [How Route 53 Determines Health](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-determining-health-of-endpoints.html)
- [Configuring Firewall Rules](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-router-firewall-rules.html)
- [Health Check Values/Parameters](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-creating-values.html)
- [Best Practices for Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-healthchecks.html)
- [Troubleshoot Unhealthy Health Checks](https://aws.amazon.com/premiumsupport/knowledge-center/route-53-fix-unhealthy-health-checks/)
- [Route 53 Pricing](https://aws.amazon.com/route53/pricing/)
- [IP Address Ranges](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-ip-addresses.html)
- [Creating DR with Route 53 (Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/)

---