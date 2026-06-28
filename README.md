# AWS Services Cheat Sheets

A collection of quick-reference cheat sheets for AWS services. Each sheet covers core concepts, architecture, CLI commands, pricing, quotas, troubleshooting, and best practices in a concise, table-driven format.

---

## Route 53

Comprehensive cheat sheets covering all major Amazon Route 53 features and components.

| Cheat Sheet | Description |
| ----------- | ----------- |
| [DNS Firewall](Route_53/Route53_DNSFirewall_CheatSheet.md) | Outbound DNS filtering for VPCs — domain lists, rule groups, actions, managed lists, advanced threat detection (DGA/tunneling), and fail-open/fail-closed modes |
| [DNSSEC](Route_53/Route53_DNSSEC_CheatSheet.md) | DNS Security Extensions signing for hosted zones — KSK/ZSK management, KMS requirements, chain of trust, enabling/disabling workflow, and CloudWatch metrics |
| [Health Checks](Route_53/Route53_HealthChecks_CheatSheet.md) | Endpoint, calculated, and CloudWatch alarm health checks — protocols, intervals, thresholds, string matching, calculated logic, and integration with routing policies |
| [Resolvers](Route_53/Route53_Resolvers_CheatSheet.md) | Hybrid DNS resolution — inbound/outbound endpoints, forwarding rules, system rules, security groups, scaling, cross-account sharing via RAM, and query logging |
| [Routing Policies](Route_53/Route53_RoutingPolicies_CheatSheet.md) | All 8 routing policies — simple, weighted, latency, failover, geolocation, geoproximity, multivalue answer, and IP-based — with health check behavior and decision guide |
| [Profiles](Route_53/Route53_Profiles_CheatSheet.md) | Centralized DNS configuration management — grouping PHZs, resolver rules, and firewall rule groups into profiles for multi-VPC/multi-account deployment via RAM |
| [Hosted Zones](Route_53/Route53_HostedZones_CheatSheet.md) | Public and private hosted zones — record types, alias records, reusable delegation sets, split-horizon DNS, cross-account PHZ association, and Traffic Flow |
| [Domains](Route_53/Route53_Domains_CheatSheet.md) | Domain registration and transfers — EPP codes, WHOIS privacy, auto-renew, transfer lock, expiration lifecycle, and supported TLDs |
| [Query Logging](Route_53/Route53_QueryLogging_CheatSheet.md) | Resolver query logging — log destinations (CloudWatch/S3/Firehose), log fields, CloudWatch Insights queries, IAM permissions, and architecture patterns |

---

## Structure

Each cheat sheet follows a consistent format:

- **Overview** — What the feature does and key points
- **Core Concepts** — Terminology and components in table format
- **How It Works** — Architecture flow diagrams
- **CLI Commands** — Ready-to-use AWS CLI examples
- **Pricing** — Cost breakdown
- **Quotas** — Service limits
- **Troubleshooting** — Common issues, causes, and fixes
- **Best Practices** — Operational recommendations
- **Useful Links** — Official AWS documentation references

---

## Usage

These cheat sheets are designed for:

- Quick reference during troubleshooting
- Exam preparation
- Onboarding new team members to Route 53 concepts
- Day-to-day operational support

---