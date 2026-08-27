# AWS Services Cheat Sheets

A collection of quick-reference cheat sheets for AWS services. Each sheet covers core concepts, architecture, CLI commands, pricing, quotas, troubleshooting, and best practices in a concise, table-driven format.

---

## Route 53

Comprehensive cheat sheets covering all major Amazon Route 53 features and components.

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Hosted Zones](Route_53/Route53_01_HostedZones_CheatSheet.md) | Public and private hosted zones — record types, alias records, reusable delegation sets, split-horizon DNS, cross-account PHZ association, and Traffic Flow |
| [02. Routing Policies](Route_53/Route53_02_RoutingPolicies_CheatSheet.md) | All 8 routing policies — simple, weighted, latency, failover, geolocation, geoproximity, multivalue answer, and IP-based — with health check behavior and decision guide |
| [03. Health Checks](Route_53/Route53_03_HealthChecks_CheatSheet.md) | Endpoint, calculated, and CloudWatch alarm health checks — protocols, intervals, thresholds, string matching, calculated logic, and integration with routing policies |
| [04. Resolvers](Route_53/Route53_04_Resolvers_CheatSheet.md) | Hybrid DNS resolution — inbound/outbound endpoints, forwarding rules, system rules, security groups, scaling, cross-account sharing via RAM, and query logging |
| [05. Profiles](Route_53/Route53_05_Profiles_CheatSheet.md) | Centralized DNS configuration management — grouping PHZs, resolver rules, and firewall rule groups into profiles for multi-VPC/multi-account deployment via RAM |
| [06. DNS Firewall](Route_53/Route53_06_DNSFirewall_CheatSheet.md) | Outbound DNS filtering for VPCs — domain lists, rule groups, actions, managed lists, advanced threat detection (DGA/tunneling), and fail-open/fail-closed modes |
| [07. DNSSEC](Route_53/Route53_07_DNSSEC_CheatSheet.md) | DNS Security Extensions signing for hosted zones — KSK/ZSK management, KMS requirements, chain of trust, enabling/disabling workflow, and CloudWatch metrics |
| [08. Query Logging](Route_53/Route53_08_QueryLogging_CheatSheet.md) | Resolver query logging — log destinations (CloudWatch/S3/Firehose), log fields, CloudWatch Insights queries, IAM permissions, and architecture patterns |
| [09. Domains](Route_53/Route53_09_Domains_CheatSheet.md) | Domain registration and transfers — EPP codes, WHOIS privacy, auto-renew, transfer lock, expiration lifecycle, and supported TLDs |

---

## WAF

Comprehensive cheat sheets covering AWS WAF (WAFv2) — Web ACLs, rules, managed protections, bot/fraud defense, logging, and operations.

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Overview & Core Concepts](WAF/WAF_01_Overview_CheatSheet.md) | What WAF protects against, core terminology, rule actions, supported resources, REGIONAL vs CLOUDFRONT scope, WCU basics, and how WAF vs Shield vs Firewall Manager fit together |
| [02. Web ACLs & Rules](WAF/WAF_02_WebACLs_and_Rules_CheatSheet.md) | Web ACL anatomy, rule structure, Action vs OverrideAction, default-action patterns, priority, labels, custom responses, associations, lock tokens, and Count-mode testing |
| [03. Rule Statements & Match Conditions](WAF/WAF_03_RuleStatements_and_MatchConditions_CheatSheet.md) | Statement types, request components, match types, text transformations, logical AND/OR/NOT, SQLi/XSS/size/geo/JSON body examples, and oversize handling |
| [04. Managed Rule Groups](WAF/WAF_04_ManagedRuleGroups_CheatSheet.md) | AWS & Marketplace managed rules — CommonRuleSet, KnownBadInputs, IP reputation, Anti-DDoS rule set, platform packs, OverrideAction, per-rule overrides, scope-down, versioning, and a baseline stack |
| [05. Rate-Based Rules](WAF/WAF_05_RateBasedRules_CheatSheet.md) | Layer 7 rate limiting — evaluation windows, aggregation keys (IP/forwarded/custom/constant), scope-down, action choices, sizing, and monitoring |
| [06. Bot Control](WAF/WAF_06_BotControl_CheatSheet.md) | Bot detection & categorization — COMMON vs TARGETED inspection levels, bot labels, verified-bot allowlisting, token/SDK integration, scope-down, and detailed gotchas |
| [07. CAPTCHA & Challenge](WAF/WAF_07_CAPTCHA_and_Challenge_CheatSheet.md) | CAPTCHA vs Challenge rule actions — token state machine, token status labels, immunity time, token domains, CORS/header caveat, SDK vs rule actions, pricing, and gotchas |
| [08. Account Takeover Prevention (ATP)](WAF/WAF_08_ATP_AccountTakeoverPrevention_CheatSheet.md) | Login protection — LoginPath/field configuration, request & response inspection, stolen-credential detection, ATP labels, and configuration gotchas |
| [09. Account Creation Fraud Prevention (ACFP)](WAF/WAF_09_ACFP_AccountCreationFraudPrevention_CheatSheet.md) | Sign-up protection — CreationPath/RegistrationPagePath, email/phone/address field mapping, response inspection, ATP vs ACFP contrast, and gotchas |
| [10. IP Sets & Regex Pattern Sets](WAF/WAF_10_IPSets_and_RegexPatternSets_CheatSheet.md) | Reusable IP sets and regex pattern sets — CIDRs, forwarded-IP config, referencing in rules, replace-on-update behavior, and quotas |
| [11. Logging & Monitoring](WAF/WAF_11_Logging_and_Monitoring_CheatSheet.md) | Log destinations (CloudWatch/S3/Firehose), `aws-waf-logs-` prefix, log filtering & redaction, log fields, CloudWatch metrics/alarms, sampled requests, and log queries |
| [12. Classic to WAFv2 Migration](WAF/WAF_12_Classic_to_WAFv2_Migration_CheatSheet.md) | Classic vs WAFv2 terminology, why migrate, migration approaches and checklist, association cutover, distinguishing the APIs, and items that don't map 1:1 |
| [13. Pricing, Quotas & Troubleshooting](WAF/WAF_13_Pricing_Quotas_Troubleshooting_CheatSheet.md) | Cost drivers and levers, WCU budget, service quotas, common API errors, operational issues, and an incident triage flow |

---

## VPC

Amazon VPC networking (excludes Transit Gateway, VPN, Cloud WAN, and VPC Lattice — see their own sections/services).

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Core, Subnets & Routing](VPC/VPC_01_Core_Subnets_Routing_CheatSheet.md) | VPC/subnet/CIDR fundamentals, reserved IPs, route tables, longest-prefix match, route targets, DNS attributes |
| [02. Security Groups & NACLs](VPC/VPC_02_SecurityGroups_and_NACLs_CheatSheet.md) | Stateful SGs vs stateless NACLs, rule evaluation, ephemeral ports, SG referencing, prefix lists |
| [03. NAT, Internet Gateway & Egress](VPC/VPC_03_NAT_IGW_and_Internet_Access_CheatSheet.md) | IGW, NAT Gateway (per-AZ HA), egress-only IGW for IPv6, public-access checklist, NAT cost pitfalls |
| [04. Endpoints & PrivateLink](VPC/VPC_04_Endpoints_and_PrivateLink_CheatSheet.md) | Gateway (S3/DynamoDB) vs interface endpoints, private DNS, endpoint policies, PrivateLink endpoint services |
| [05. VPC Peering](VPC/VPC_05_Peering_CheatSheet.md) | 1:1 peering, non-transitivity, CIDR overlap, both-side routing, cross-account/Region, peering vs TGW |
| [06. Flow Logs](VPC/VPC_06_FlowLogs_CheatSheet.md) | Traffic metadata capture, fields (incl. tcp-flags/pkt-srcaddr), destinations, not-logged traffic, Athena queries |
| [07. Network Analysis & Visibility Tools](VPC/VPC_07_NetworkAnalysis_Tools_CheatSheet.md) | Traffic Mirroring (sessions/targets/filters, UDP 4789, MTU/Nitro caveats), Reachability Analyzer (path debug), Network Access Analyzer (scope/findings audit) |
| [08. Block Public Access & DHCP Option Sets](VPC/VPC_08_BlockPublicAccess_and_DHCP_CheatSheet.md) | VPC BPA modes (bidirectional/ingress) + exclusions + NFW/GA behavior; DHCP option sets (custom DNS, AmazonProvidedDNS + DNS Firewall inspection caveat, immutability) |

---

## Transit Gateway

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Overview](TGW/TGW_01_Overview_CheatSheet.md) | Regional routing hub — concepts, topologies, segmentation, appliance mode, pricing, quotas |
| [02. Attachments](TGW/TGW_02_Attachments_CheatSheet.md) | VPC/VPN/DX-GW/peering/Connect attachments, one-subnet-per-AZ, bandwidth and per-flow limits |
| [03. Routing (Associations & Propagations)](TGW/TGW_03_Routing_CheatSheet.md) | TGW route tables, association vs propagation, static/blackhole routes, segmentation patterns |
| [04. Peering](TGW/TGW_04_Peering_CheatSheet.md) | Cross-Region/account TGW peering, static-route requirement, non-transitivity |
| [05. Sharing & Multi-Account](TGW/TGW_05_Sharing_and_MultiAccount_CheatSheet.md) | RAM sharing, delegated ownership, auto-accept vs manual, owner-managed routing |
| [06. Monitoring & Flow Logs](TGW/TGW_06_Monitoring_and_FlowLogs_CheatSheet.md) | CloudWatch metrics, drop reasons, TGW Flow Logs, Network Manager, Route Analyzer |
| [07. Multicast](TGW/TGW_07_Multicast_CheatSheet.md) | Multicast domains, IGMPv2 vs static source/member models, subnet association, IGMP query/SG/NACL rules, Nitro/MTU/attachment-type caveats |

---

## Network Firewall

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Overview](Network_Firewall/NFW_01_Overview_CheatSheet.md) | Managed stateful firewall/IPS — concepts, stateless vs stateful, deployment overview, pricing, quotas |
| [02. Stateless Rule Groups](Network_Firewall/NFW_02_StatelessRuleGroups_CheatSheet.md) | 5-tuple rules, priority, actions, `aws:forward_to_sfe`, capacity, fragment defaults |
| [03. Stateful Rules & Default Actions](Network_Firewall/NFW_03_StatefulRules_and_DefaultActions_CheatSheet.md) | Stateful rule types, domain lists, rule order (strict vs default), all default actions incl. application-layer, and when to use which |
| [04. Suricata Rules](Network_Firewall/NFW_04_Suricata_CheatSheet.md) | Suricata syntax, actions, rule variables, flow/app-layer keywords, examples, compatibility caveats |
| [05. Firewall Policy](Network_Firewall/NFW_05_FirewallPolicy_CheatSheet.md) | Policy contents, stateless/stateful references, defaults, rule order, protections |
| [06. Deployment Models](Network_Firewall/NFW_06_DeploymentModels_CheatSheet.md) | Distributed, centralized (inspection VPC + TGW), TGW-attached (native), decentralized ingress — with a when-to-use decision guide |
| [07. Associations](Network_Firewall/NFW_07_Associations_CheatSheet.md) | Firewall–policy, subnet/endpoint, VPC endpoint associations, TGW attachment, RAM sharing, and container associations (ECS/EKS dynamic IP sets) |
| [08. Symmetric Routing & Troubleshooting](Network_Firewall/NFW_08_SymmetricRouting_and_Troubleshooting_CheatSheet.md) | Asymmetric-routing detection test, stream exception policy, GWLB 350s idle timeout, ReceivedPackets, constrained AZ, fail-close/scaling |
| [09. TLS Inspection](Network_Firewall/NFW_09_TLS_Inspection_CheatSheet.md) | Decrypt/inspect TLS, inbound vs outbound, ACM certs, scope, MITM/pinning caveats |
| [10. Logging & Monitoring](Network_Firewall/NFW_10_Logging_and_Monitoring_CheatSheet.md) | ALERT/FLOW/TLS logs, CloudWatch metrics, Athena queries |

---

## Shield Advanced

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Overview](Shield_Advanced/Shield_01_Overview_CheatSheet.md) | Standard vs Advanced, subscription model, feature summary |
| [02. Protections & Resources](Shield_Advanced/Shield_02_Protections_and_Resources_CheatSheet.md) | Protectable resource types, per-ARN protections, protection groups |
| [03. App-Layer Mitigation & WAF](Shield_Advanced/Shield_03_AppLayer_Mitigation_and_WAF_CheatSheet.md) | Automatic application-layer mitigation, Count vs Block, WAF included |
| [04. Detection & Health Checks](Shield_Advanced/Shield_04_Detection_and_HealthChecks_CheatSheet.md) | Baselining, health-based detection, attack diagnostics, `DDoSDetected` |
| [05. SRT & Response](Shield_Advanced/Shield_05_SRT_and_Response_CheatSheet.md) | Shield Response Team, proactive engagement, SRT access, emergency contacts |
| [06. Cost Protection & Pricing](Shield_Advanced/Shield_06_CostProtection_and_Pricing_CheatSheet.md) | Subscription fee (per org), DDoS cost-protection credits, DTO fees |

---

## Shield Network Security Director (NSD)

A separate AWS Shield capability (distinct from Shield Advanced), currently in **public preview** — subject to change.

| Cheat Sheet | Description |
| ----------- | ----------- |
| [Network Security Director (NSD)](Shield_NSD/Shield_NetworkSecurityDirector_CheatSheet.md) | Network posture analysis (preview) — resource discovery, findings by resource type, severity, topology, remediation, Organizations + Security Hub + Amazon Q integration |

---

## Firewall Manager

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Overview](FMS/FMS_01_Overview_CheatSheet.md) | Prerequisites (Org, delegated admin, Config), all policy types (incl. Network ACL and third-party Palo Alto/Fortigate), auto-remediation/drift |
| [02. WAF Policy](FMS/FMS_02_WAF_Policy_CheatSheet.md) | Managed Web ACLs, pre/post-process rule groups, drift, logging bucket statements |
| [03. Shield Policy](FMS/FMS_03_Shield_Policy_CheatSheet.md) | Org-wide Shield Advanced enrollment, subscription requirement |
| [04. Security Group Policies](FMS/FMS_04_SecurityGroup_Policies_CheatSheet.md) | Common / content-audit / usage-audit modes, guardrails, cleanup |
| [05. Network & DNS Firewall Policies](FMS/FMS_05_NetworkFirewall_and_DNSFirewall_Policies_CheatSheet.md) | Distributed/centralized Network Firewall + Resolver DNS Firewall deployment |
| [06. Compliance & Remediation](FMS/FMS_06_Compliance_and_Remediation_CheatSheet.md) | Config-based evaluation, auto-remediation, drift, throttling, notifications |

---

## IPAM

| Cheat Sheet | Description |
| ----------- | ----------- |
| [01. Overview](IPAM/IPAM_01_Overview_CheatSheet.md) | IP planning service — hierarchy, capabilities, tiers, pricing, quotas |
| [02. Scopes & Pools](IPAM/IPAM_02_Scopes_and_Pools_CheatSheet.md) | Public/private scopes, nested pool hierarchy, locale, allocation rules |
| [03. Allocations & Auto-Allocation](IPAM/IPAM_03_Allocations_CheatSheet.md) | VPC/subnet/manual allocations, auto-allocation, closed-account/stuck CIDRs |
| [04. BYOIP](IPAM/IPAM_04_BYOIP_CheatSheet.md) | Bring-your-own IPv4/IPv6, ROA + signed authorization, advertising control |
| [05. Monitoring & Resource Discovery](IPAM/IPAM_05_Monitoring_and_ResourceDiscovery_CheatSheet.md) | Utilization metrics, overlap detection, allocation history, org-wide discovery, and free Public IP Insights (public IPv4 audit + unused-EIP cleanup) |
| [06. Sharing & Multi-Account](IPAM/IPAM_06_Sharing_and_MultiAccount_CheatSheet.md) | Delegated admin, RAM pool sharing, Organizations integration |

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
- Onboarding new team members to AWS service concepts
- Day-to-day operational support

---