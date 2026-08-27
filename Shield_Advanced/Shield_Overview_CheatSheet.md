# AWS Shield Advanced - Cheat Sheet

## Overview

**AWS Shield Advanced** is a paid, managed DDoS protection service that builds on the free **Shield Standard**. It adds enhanced Layer 3/4 and Layer 7 attack detection and mitigation, near-real-time attack visibility, **DDoS cost protection** (credits for scaling during attacks), automatic application-layer mitigation via WAF, and access to the **Shield Response Team (SRT)**.

**Key point:** Shield Advanced is a **subscription** (1-year commitment, monthly fee) that applies **per organization** and protects **specified, registered resources**. WAF is included at no extra WAF fee for protected resources.

---

## Shield Standard vs Advanced

| Capability                          | Shield Standard (free) | Shield Advanced (paid) |
| ----------------------------------- | ---------------------- | ---------------------- |
| L3/L4 DDoS protection               | Yes (automatic)        | Yes (enhanced)         |
| L7 (application) protection         | No                     | Yes (with WAF)         |
| Attack diagnostics/visibility       | Limited                | Detailed, near real-time |
| DDoS cost protection (credits)      | No                     | Yes                    |
| Shield Response Team (SRT)          | No                     | Yes                    |
| Proactive engagement                | No                     | Yes (optional)         |
| WAF fees for protected resources    | N/A                    | Included               |
| Global threat dashboard             | No                     | Yes                    |

---

## Protected Resource Types

| Resource                                 | Notes                                            |
| ---------------------------------------- | ------------------------------------------------ |
| **CloudFront distributions**             | Global; L3/4 + L7                                |
| **Route 53 hosted zones**                | DNS protection                                   |
| **Global Accelerator accelerators**      | L3/4                                             |
| **Application Load Balancer (ALB)**      | L3/4 + L7 (via WAF)                              |
| **Network / Classic Load Balancer**      | L3/4                                             |
| **Elastic IPs (EC2/NLB)**                | L3/4                                             |

> You must **explicitly add** each resource to Shield Advanced protection; subscribing alone doesn't protect anything.

---

## Key Features

| Feature                         | Description                                                        |
| ------------------------------- | ------------------------------------------------------------------ |
| **Automatic app-layer mitigation** | Shield can auto-create/manage WAF rate/rule mitigations during L7 attacks |
| **Health-based detection**      | Associate Route 53 health checks so Shield knows app health, reducing false positives |
| **DDoS cost protection**        | Service credits for usage spikes (e.g., scaling, data transfer) caused by a covered DDoS attack |
| **Shield Response Team (SRT)**  | Experts who help during/after attacks (contact or proactive engagement) |
| **Proactive engagement**        | SRT contacts you when health checks indicate an attack             |
| **Protection groups**           | Group resources for aggregated detection/reporting                 |

---

## How It Works

```
Traffic to a protected resource
        |
        v
Shield Standard (always-on L3/4 mitigation at the edge)
        |
        v
Shield Advanced detection (baselining + health checks)
        |
        ├── L3/4 attack → enhanced automatic mitigation
        └── L7 attack   → WAF rules (automatic app-layer mitigation or your rules)
        |
        v
Attack visibility in the Shield console + SRT engagement available
```

---

## CLI

```bash
# Subscribe (one-time; 1-year commitment)
aws shield create-subscription

# Protect a resource (e.g., a CloudFront distribution or EIP)
aws shield create-protection \
  --name "prod-cf" \
  --resource-arn arn:aws:cloudfront::123456789012:distribution/EABCD

# Associate a Route 53 health check with a protection (health-based detection)
aws shield associate-health-check \
  --protection-id <id> --health-check-arn arn:aws:route53:::healthcheck/abc

# Enable proactive engagement + set SRT contacts
aws shield associate-proactive-engagement-details \
  --emergency-contact-list EmailAddress=soc@example.com,PhoneNumber=+15551234567
aws shield enable-proactive-engagement

# List attacks and protections
aws shield list-attacks
aws shield list-protections
aws shield describe-attack --attack-id <id>
```

---

## Pricing

| Item                         | Cost                                              |
| ---------------------------- | ------------------------------------------------- |
| Subscription                 | ~$3,000 / month, **1-year commitment**, per organization (consolidated billing) |
| Data transfer out (during protection) | Additional per-GB fees on protected resources |
| WAF for protected resources  | Included (no separate WAF Web ACL/rule/request fees) |
| DDoS cost protection         | Credits offset attack-driven usage spikes         |

> The monthly fee applies **once per consolidated-billing family** (not per account). WAF usage on protected resources is included, which offsets some cost.

---

## Gotchas & Caveats

1. **Subscription is a 1-year commitment** billed monthly — you can't casually turn it on for a week.
2. **The fee is per organization (consolidated billing family), not per account** — subscribe in the management account context; all member accounts are covered.
3. **You must add each resource to protection** — subscribing does nothing until you create protections for specific ARNs.
4. **Only specific resource types are supported** (CloudFront, R53, Global Accelerator, ALB/NLB/CLB, EIP) — you can't protect arbitrary services or raw EC2 without an EIP/LB.
5. **DDoS cost protection is a credit request, not automatic waiver** — you request credits for covered spikes; it's not blanket free scaling.
6. **L7 protection means WAF** — application-layer mitigation happens through WAF rules; without WAF association it's L3/4 only.
7. **Health-based detection needs Route 53 health checks associated** — without them, detection is less accurate and proactive engagement is limited.
8. **Proactive engagement requires configured SRT contacts** and eligible resources — set contacts before an incident.
9. **Automatic application-layer mitigation can create/modify WAF rules** — understand what it changes so you don't fight it during an incident.
10. **Global Accelerator/CloudFront are global**; regional resources (ALB/EIP) are protected per Region — plan protections accordingly.
11. **Shield doesn't stop all attacks by itself** — you still need good WAF rules, rate-based rules, and architecture (edge, autoscaling).
12. **Deregistering/deleting a protected resource** removes its protection silently — audit protections after infra changes.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Resource not protected during attack        | Never added to Shield protections               | Create a protection for the resource ARN                  |
| L7 attack not mitigated                     | No WAF / no auto app-layer mitigation           | Associate WAF; enable automatic app-layer mitigation      |
| Too many false-positive mitigations         | No app health signal                            | Associate Route 53 health checks                          |
| SRT didn't proactively engage               | Proactive engagement not enabled / no contacts  | Enable proactive engagement; set emergency contacts       |
| Unexpected costs during spike               | Attack-driven scaling                           | Request DDoS cost-protection credits with attack evidence |
| Can't protect an EC2 instance               | No EIP/LB in front                              | Attach an EIP or front with ALB/NLB                       |

---

## Best Practices

1. **Subscribe at the org level** and protect resources across member accounts.
2. **Register all internet-facing resources** (CloudFront, ALB, NLB, EIP, R53) that need coverage.
3. **Associate Route 53 health checks** for accurate, health-based detection.
4. **Pair with WAF** (managed rules + rate-based rules) for real L7 defense.
5. **Enable automatic application-layer mitigation** and understand the WAF rules it manages.
6. **Configure SRT contacts and proactive engagement** before you need them.
7. **Use protection groups** to aggregate detection across related resources.
8. **Keep attack evidence** to support DDoS cost-protection credit requests.
9. **Architect for resilience** (edge caching, autoscaling, multi-AZ) — Shield complements, not replaces, good design.

---

## Useful Links

- [AWS Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
- [Getting started with Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/getting-started-ddos.html)
- [Resources you can protect](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
- [Automatic application-layer DDoS mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-automatic-app-layer-response.html)
- [Health-based detection](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-health-checks.html)
- [Shield Response Team (SRT)](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt.html)
- [DDoS cost protection & pricing](https://aws.amazon.com/shield/pricing/)

---
