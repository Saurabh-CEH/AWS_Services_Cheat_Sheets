# AWS Shield Advanced - Protections & Resources Cheat Sheet

## Overview

A **protection** is Shield Advanced's per-resource enrollment. Subscribing to Shield Advanced does nothing until you create protections for specific resource ARNs. Protections enable enhanced detection, DDoS cost protection, and (for supported types) application-layer defenses via WAF.

**Key point:** Protection is **per resource ARN** — you must explicitly add each internet-facing resource, and you can group related resources with **protection groups** for aggregated detection.

---

## Protectable Resource Types

| Resource                                 | Layer | Notes                                            |
| ---------------------------------------- | ----- | ------------------------------------------------ |
| **CloudFront distribution**              | L3/4 + L7 | Global; best place for L7 + edge protection   |
| **Route 53 hosted zone**                 | L3/4  | DNS infrastructure protection                    |
| **Global Accelerator accelerator**       | L3/4  | Anycast edge                                     |
| **Application Load Balancer (ALB)**      | L3/4 + L7 | L7 via associated WAF                          |
| **Network Load Balancer (NLB)**          | L3/4  | Regional                                         |
| **Classic Load Balancer (CLB)**          | L3/4  | Regional                                         |
| **Elastic IP (EC2/NLB)**                 | L3/4  | Protects the EIP-associated resource             |

> To protect an EC2 instance, front it with an **EIP** or a **load balancer** — you can't protect a bare instance.

---

## Protection Groups

Group multiple protected resources so Shield evaluates them together (better detection for resources that share traffic patterns) and reports aggregated events.

| Setting              | Detail                                                        |
| -------------------- | ------------------------------------------------------------- |
| **Aggregation**      | SUM / MEAN / MAX across the group's resources                 |
| **Pattern**          | ALL resources, resources by type, or an arbitrary set         |
| **Benefit**          | Improved detection accuracy; consolidated reporting            |

---

## How Protection Works

```
Subscribe to Shield Advanced (org-level, 1-year)
        |
        v
Create a PROTECTION per resource ARN (CloudFront, ALB, EIP, R53, GA, NLB/CLB)
        |
        ├── (optional) associate a Route 53 health check for health-based detection
        ├── (optional) group resources into a protection group
        └── (ALB/CloudFront) associate WAF for L7 + automatic app-layer mitigation
        |
        v
Enhanced detection + DDoS cost protection now apply to that resource
```

---

## CLI

```bash
# Protect a resource
aws shield create-protection \
  --name "prod-alb" \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/prod/abc

# List protections
aws shield list-protections
aws shield describe-protection --protection-id <id>

# Create a protection group (aggregate by type)
aws shield create-protection-group \
  --protection-group-id prod-web \
  --aggregation SUM --pattern BY_RESOURCE_TYPE --resource-type APPLICATION_LOAD_BALANCER

# Remove a protection
aws shield delete-protection --protection-id <id>
```

---

## Gotchas & Caveats

1. **Subscribing ≠ protecting** — you must create a protection for each resource ARN; nothing is protected automatically.
2. **Only specific resource types are supported** — CloudFront, R53, Global Accelerator, ALB/NLB/CLB, EIP. Bare EC2, API Gateway (directly), etc. are not directly protectable.
3. **Protect EC2 via EIP or load balancer** — attach an EIP or front with an LB, then protect that.
4. **L7 protection means CloudFront/ALB + WAF** — NLB/CLB/EIP get L3/4 only.
5. **Deleting/replacing a resource silently drops its protection** — re-create protection after infra changes; audit periodically.
6. **Regional vs global** — CloudFront/Global Accelerator are global; ALB/NLB/EIP are per-Region. Protect in each Region as needed.
7. **Protection groups improve detection but need sensible aggregation** — wrong SUM/MEAN/MAX or pattern can dilute signal.
8. **Health-based detection needs a Route 53 health check associated** to the protection (see the Detection sheet).
9. **Association with WAF is separate** — protecting the resource doesn't auto-create WAF rules; associate a Web ACL for L7.
10. **Quota on protected resources** exists — very large fleets may need increases.
11. **Cross-account** — protections are created in the account owning the resource; the Shield subscription is org-level.
12. **Global Accelerator protection** covers the accelerator; protect the origin resources too if directly exposed.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Resource not defended during attack         | No protection created for it                    | `create-protection` for the resource ARN                  |
| Can't protect EC2 instance                  | Bare instance unsupported                       | Attach an EIP or front with an LB, protect that           |
| L7 attack not mitigated                     | NLB/EIP only (no WAF) or WAF not associated      | Use CloudFront/ALB + associate WAF                        |
| Protection disappeared                      | Underlying resource replaced/deleted            | Recreate protection; audit after changes                  |
| Detection noisy/inaccurate                  | No protection group / no health check           | Group related resources; associate health checks          |
| Quota exceeded                              | Too many protected resources                    | Request a quota increase                                  |

---

## Best Practices

1. **Enroll all internet-facing resources** you care about (CloudFront, ALB, NLB, EIP, R53, GA).
2. **Front EC2 with EIP/LB** and protect that.
3. **Use CloudFront/ALB + WAF** for anything needing L7 defense.
4. **Create protection groups** for related resources to sharpen detection.
5. **Associate Route 53 health checks** for health-based detection.
6. **Audit protections after infra changes** so nothing silently loses coverage.
7. **Protect in every Region** where you run regional resources.
8. **Automate protection creation** in your provisioning (IaC) so new resources are enrolled.

---

## Useful Links

- [Resources you can protect with Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
- [Adding Shield Advanced protection](https://docs.aws.amazon.com/waf/latest/developerguide/configure-new-protection.html)
- [Protection groups](https://docs.aws.amazon.com/waf/latest/developerguide/manage-protection-group.html)
- [Getting started](https://docs.aws.amazon.com/waf/latest/developerguide/getting-started-ddos.html)

---
