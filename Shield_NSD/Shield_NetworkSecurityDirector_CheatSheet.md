# AWS Shield Network Security Director (NSD) - Cheat Sheet

## Overview

**AWS Shield network security director (NSD)** is a Shield capability (**public preview**) that performs **network posture analysis**: it discovers resources across your accounts, maps how they connect, evaluates their security configuration against **AWS best practices + threat intelligence**, and produces **findings** with **remediation recommendations**. Think of it as "what in my network is exposed or missing protection, and how do I fix it" — complementary to Shield Advanced's DDoS mitigation.

**Key point:** NSD is **read/analyze-and-recommend**, not a mitigation engine. It surfaces misconfigurations (open security groups, missing WAF rules, unassociated Web ACLs, DDoS/bot activity) and tells you how to remediate. It's **Organizations-based** and viewed by a **delegated administrator**.

---

## Core Concepts

| Concept             | Description                                                                 |
| ------------------- | --------------------------------------------------------------------------- |
| **Resources**       | Compute (EC2), Networking (ALB, API Gateway, CloudFront, VPC subnets, ENIs), Security (WAF Web ACLs, security groups, NACLs) that handle app traffic |
| **Findings**        | Alerts about missing/misconfigured security, each with a **severity**       |
| **Severity**        | NONE / INFORMATIONAL / LOW / MEDIUM / HIGH / CRITICAL — a resource's level = its **most severe** finding |
| **Composite severity** | A resource's overall level across all its findings (highest wins)        |
| **Network topology**| Visual map of resource connections, internet exposure, tag relationships    |
| **Network analysis**| The scan that (re)generates findings and topology                           |

---

## How It Works

```
Enable NSD (Organizations, via delegated admin)
        │
        ▼
Network analysis discovers resources across accounts/regions
        │
        ├── Maps connectivity → network topology (internet exposure, edges)
        ├── Evaluates each resource vs best practices + threat intel
        ▼
Findings (per resource, with severity)
        │
        ▼
Remediation recommendations (+ doc links)  ──► optional Amazon Q Developer analysis
        │
        ▼
Findings also flow to AWS Security Hub
```

- View on the **network security director dashboard** (console) — Regions widget, Accounts widget, Account & topology explorer, Network topology.
- **Delegated administrator credentials are required** to view resources/findings.
- Topology shows only the **first 100 connected resources** (sorted by severity) from the selected resource.

---

## Findings by Resource Type

| Resource type            | Representative findings                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------- |
| **Application Load Balancer** | CloudFront origin also internet-accessible w/o CloudFront protections; WAF missing bot/scraper rules; **DDoS activity detected**; no firewall attached; WAF missing all rules; WAF missing key AWS Managed Rules (IP Reputation / Common / Bad Inputs) |
| **API Gateway**          | WAF missing bot/scraper rules; no firewall attached; WAF missing all rules; WAF missing key AWS Managed Rules |
| **CloudFront**           | WAF missing bot/scraper rules; DDoS activity detected; no firewall attached; WAF missing all rules; WAF missing key AWS Managed Rules |
| **EC2 instance**         | Unrestricted inbound `0.0.0.0/0` on all ports / to **RDP 3389** / to **SSH 22**; unrestricted outbound; no firewall attached; CloudFront origin internet-accessible; WAF gaps |
| **Security group**       | Unrestricted inbound on all ports / RDP 3389 / SSH 22; unrestricted outbound on all ports |
| **NACL**                 | Unrestricted inbound on all ports / RDP 3389 / SSH 22; unrestricted outbound on all ports |
| **WAF Web ACL**          | **Bot activity detected**; missing bot/scraper rules; **Web ACL not associated with any resource**; missing all rules; missing key AWS Managed Rules |

> Findings blend **config gaps** (open SGs/NACLs, unassociated Web ACL, missing managed rules) with **observed activity** (DDoS/bot activity detected).

---

## Using NSD (workflow)

1. **Set up** NSD from the delegated admin account (Organizations); enable the regions you care about.
2. **Run/allow a network analysis** to populate findings + topology.
3. On the **Dashboard**, use the **Regions** widget to spot the worst regions, then the **Accounts** widget (sorted by composite severity).
4. Open the **Account & topology explorer** → sort **Resources by severity** (highest first).
5. Open a high-severity resource → review its **Findings** → expand **Remediation recommendations** and follow the steps / doc links.
6. Explore the **Network topology** to understand exposure and edges (select an edge to learn the relationship).
7. Optionally hand off to **Amazon Q Developer** for deeper security analysis.

---

## Integrations

| Integration              | What it enables                                                            |
| ------------------------ | -------------------------------------------------------------------------- |
| **AWS Organizations**    | Org-wide analysis; managed via **Network Security Director policies**; delegated admin |
| **AWS Security Hub**     | NSD findings are available in Security Hub for centralized security posture |
| **Amazon Q Developer**   | Natural-language analysis of your NSD findings/posture                     |

---

## Gotchas & Caveats

1. **Public preview** — features/behavior subject to change; validate before relying on it operationally.
2. **NSD analyzes and recommends; it does NOT mitigate** — it won't fix SGs or add WAF rules for you (that's your remediation, or Firewall Manager).
3. **Delegated administrator required** to view findings/topology — set up Organizations + delegated admin first.
4. **Topology caps at 100 connected resources** (severity-sorted) per selected resource — large blast radii aren't fully drawn.
5. **Composite severity = worst finding** — a "Medium" resource may still have several lower findings to work through.
6. **Region-scoped** — enable and review each region; the Regions widget flags where you're missing coverage.
7. **Findings overlap other tools** — "missing key AWS Managed Rules," "Web ACL not associated" etc. also relate to WAF/FMS; use NSD to prioritize, FMS to enforce org-wide.
8. **Not the same as Shield Advanced DDoS mitigation** — NSD is posture/visibility; Shield Advanced (and the WAF Anti-DDoS rule group) do the mitigation.

---

## Troubleshooting

| Issue                                   | Cause                                       | Fix                                                    |
| --------------------------------------- | ------------------------------------------- | ------------------------------------------------------ |
| Can't see findings/topology             | Not signed in as delegated admin            | Use delegated administrator credentials                |
| No resources/findings appear            | Analysis not run / region not enabled       | Enable the region; run/await a network analysis        |
| Topology incomplete for a big resource  | 100-connection display cap                  | Investigate connected resources individually           |
| Findings not in Security Hub            | Security Hub integration not enabled        | Enable Security Hub + the NSD integration              |
| Findings persist after fixing           | Analysis not re-run                         | Re-run analysis; findings reflect the **last** scan    |

---

## Best Practices

1. **Onboard via Organizations + a delegated admin** so analysis spans the whole org.
2. **Triage by composite severity** (Critical/High first), region by region.
3. **Feed findings into Security Hub** for centralized posture and ticketing.
4. **Remediate config gaps with Firewall Manager** (missing WAF rules, unassociated Web ACLs) org-wide, not one account at a time.
5. **Close obvious exposure first** — unrestricted `0.0.0.0/0` on SSH/RDP/all-ports is the highest-value quick win.
6. **Re-run analysis after remediation** to confirm findings clear.
7. **Pair NSD (visibility) with Shield Advanced + WAF Anti-DDoS (mitigation)** for both posture and active defense.

---

## Useful Links

- [Network security director key concepts](https://docs.aws.amazon.com/waf/latest/developerguide/nsd-concepts.html)
- [Exploring resources and findings](https://docs.aws.amazon.com/waf/latest/developerguide/nsd-findings.html)
- [Use cases](https://docs.aws.amazon.com/waf/latest/developerguide/nsd-use-cases.html)
- [Remediation steps](https://docs.aws.amazon.com/waf/latest/developerguide/nsd-remediation-steps.html)
- [Network Security Director Organizations policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_network_security_director.html)
- [NSD findings in Security Hub (announcement)](https://aws.amazon.com/about-aws/whats-new/2026/03/network-security-director-findings/)
- [Shield network security director (preview announcement)](https://aws.amazon.com/about-aws/whats-new/2025/06/aws-shield-network-security-director-preview/)

---

*Sources: AWS official documentation. Content was rephrased for compliance with licensing restrictions. NSD is in public preview; always refer to the official AWS documentation for the most current information.*
