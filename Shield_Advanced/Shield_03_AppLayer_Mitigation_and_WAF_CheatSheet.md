# AWS Shield Advanced - Application-Layer Mitigation & WAF Cheat Sheet

## Overview

Shield Advanced's **Layer 7 (application-layer) defense** works through **AWS WAF**. For protected CloudFront distributions and ALBs, Shield can provide **automatic application-layer DDoS mitigation** — creating and managing WAF rate-based rules during an attack — and gives you WAF at no extra WAF fee on protected resources.

**Key point:** L7 protection = WAF. Without a Web ACL associated with the resource (and automatic mitigation enabled or your own rules), Shield Advanced only covers L3/4.

---

## Core Concepts

| Concept                              | Description                                                    |
| ------------------------------------ | -------------------------------------------------------------- |
| **Automatic app-layer mitigation**  | Shield auto-creates/manages WAF rules during detected L7 attacks |
| **Mitigation modes**                 | **Count** (observe) or **Block** (enforce)                     |
| **WAF included**                     | No separate WAF Web ACL/rule/request fees on protected resources |
| **Baseline**                         | Shield learns normal traffic to distinguish attack from legit  |
| **Your own WAF rules**               | Rate-based rules, managed rule groups, and custom rules still apply |

---

## How Automatic Mitigation Works

```
Protected CloudFront/ALB with an associated Web ACL + automatic mitigation enabled
        |
        v
Shield detects an application-layer DDoS (baseline deviation + health signals)
        |
        ├── mode = Count → creates WAF rules that COUNT the attack pattern (observe)
        └── mode = Block → creates WAF rules that BLOCK the attack pattern
        |
        v
Rules are managed by Shield for the duration; removed/adjusted as the attack subsides
```

- Shield places its rules in a dedicated spot in your Web ACL and manages their lifecycle.
- Health-based detection (Route 53 health checks) reduces false positives.

---

## Enabling

```bash
# Enable automatic application-layer DDoS mitigation for a protected resource
aws shield enable-application-layer-automatic-response \
  --resource-arn arn:aws:cloudfront::123456789012:distribution/EABCD \
  --action '{"Block":{}}'          # or {"Count":{}}

# Update the action (switch Count <-> Block)
aws shield update-application-layer-automatic-response \
  --resource-arn arn:... --action '{"Count":{}}'

# Disable
aws shield disable-application-layer-automatic-response --resource-arn arn:...
```

> The resource must have a WAF **Web ACL associated** before you enable automatic mitigation.

---

## Relationship to Your WAF Rules

| Layer                          | Who manages it                                   |
| ------------------------------ | ------------------------------------------------ |
| Managed rule groups, custom rules | You                                            |
| Rate-based rules               | You (recommended baseline for L7 floods)         |
| Automatic mitigation rules     | **Shield** (created/removed during attacks)      |

> Keep your own **rate-based rules** and **managed rule groups** in place — automatic mitigation is an addition, not a replacement, for good WAF hygiene.

---

## Gotchas & Caveats

1. **L7 defense requires WAF** — associate a Web ACL with the CloudFront/ALB or you only get L3/4.
2. **Automatic mitigation needs a Web ACL associated first** — enabling it without one fails.
3. **Automatic mitigation creates/manages WAF rules in your Web ACL** — don't be surprised by rules you didn't author; don't hand-edit Shield-managed rules.
4. **Start in Count mode** — going straight to Block risks false positives on legitimate spikes; validate first.
5. **Health checks improve accuracy** — without associated Route 53 health checks, detection is less precise and may over/under-react.
6. **Baselines take time** — Shield needs to learn normal traffic; brand-new resources have weaker baselines initially.
7. **WCU/rule budget** — Shield's rules consume Web ACL capacity; ensure headroom.
8. **Only CloudFront and ALB** support automatic application-layer mitigation — NLB/EIP/Global Accelerator are L3/4.
9. **Included WAF fees apply only to protected resources** — WAF on non-protected resources bills normally.
10. **You still design the Web ACL** — automatic mitigation targets the specific attack; broad protection (bad inputs, bots) is your rules' job.
11. **Turning it off mid-attack** removes Shield's rules — coordinate during incidents.
12. **Reviewing what Shield did** — inspect the Web ACL and Shield events after an attack to understand the mitigations applied.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| L7 attack not mitigated                     | No WAF / automatic mitigation not enabled       | Associate a Web ACL; enable automatic app-layer response  |
| Can't enable automatic mitigation           | No Web ACL associated                           | Associate a Web ACL first                                 |
| Legitimate traffic blocked during spike     | Block mode + no health signal                   | Use Count mode; associate health checks; tune             |
| Unknown WAF rules appeared                   | Shield-managed automatic mitigation rules       | Expected — don't hand-edit; review Shield events          |
| WCU limit issues                            | Shield rules + your rules exceed budget          | Free capacity or request WCU increase                     |
| No mitigation on NLB/EIP                     | L3/4 only                                        | Front with CloudFront/ALB for L7                          |

---

## Best Practices

1. **Associate a WAF Web ACL** with every protected CloudFront/ALB.
2. **Keep your own rate-based rules and managed rule groups** as the baseline.
3. **Enable automatic app-layer mitigation in Count first**, validate, then switch to Block.
4. **Associate Route 53 health checks** so mitigation is health-aware.
5. **Leave WCU headroom** for Shield-managed rules.
6. **Don't hand-edit Shield-managed rules**; adjust via mode/config.
7. **Review Shield events + Web ACL after incidents** to learn and tune.

---

## Useful Links

- [Automatic application-layer DDoS mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-automatic-app-layer-response.html)
- [Shield Advanced and WAF](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-manage-protected-resources.html)
- [Responding to DDoS events](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-responding.html)
- [WAF rate-based rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html)

---
