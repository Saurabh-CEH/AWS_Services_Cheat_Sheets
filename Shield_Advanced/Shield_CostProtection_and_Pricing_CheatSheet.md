# AWS Shield Advanced - Cost Protection & Pricing Cheat Sheet

## Overview

Shield Advanced is a **subscription** with a monthly fee and a **1-year commitment**, applied **per consolidated-billing family (organization)**. It includes **DDoS cost protection** — service credits that offset usage spikes caused by a covered DDoS attack — and includes WAF fees on protected resources.

**Key point:** The subscription fee is charged **once per organization** (not per account), and DDoS cost protection is a **credit request** (with attack evidence), not an automatic waiver.

---

## Pricing Structure

| Component                         | Detail                                                        |
| --------------------------------- | ------------------------------------------------------------- |
| **Subscription fee**              | ~$3,000/month, **1-year commitment**, per organization        |
| **Data transfer out (DTO) fees**  | Additional per-GB DTO charges on protected resources (CloudFront, ALB, etc.) |
| **Included WAF**                  | No separate WAF Web ACL/rule/request fees on protected resources |
| **DDoS cost protection**          | Credits for attack-driven usage spikes (on request)           |

> The fee is **per payer/consolidated-billing family** — subscribing once covers all member accounts. WAF being included offsets some cost if you'd otherwise pay for WAF on those resources.

---

## DDoS Cost Protection

Covers scaling/usage spikes directly caused by a covered DDoS attack on protected resources, for services such as:

| Service                     | Covered spike example                              |
| --------------------------- | -------------------------------------------------- |
| **CloudFront**              | Surge in data transfer / requests                  |
| **Route 53**               | Surge in DNS queries                               |
| **ALB / ELB**              | Scaling / LCU surge                                |
| **EC2 (behind protection)** | Autoscaling / data transfer during attack          |
| **Global Accelerator**      | Data transfer surge                                |

### How to claim

```
1. Attack occurs on a PROTECTED resource
2. Usage/bill spikes due to the attack
3. Open a support case with attack evidence (Shield attack ID, timeframe, affected resources)
4. AWS reviews and issues service credits for the covered spike
```

---

## Gotchas & Caveats

1. **1-year commitment** — the subscription isn't month-to-month; you commit for a year.
2. **Fee is per organization (consolidated billing), not per account** — don't multiply the fee across member accounts; subscribe once for the family.
3. **Cost protection is a credit request, not automatic** — you must open a case with attack evidence; it's not a blanket free-scaling guarantee.
4. **Only covers spikes on protected resources caused by a covered attack** — unprotected resources or non-attack spikes aren't covered.
5. **Data transfer out fees still apply** — protected resources incur additional DTO charges (this is separate from the subscription fee).
6. **Included WAF applies only to protected resources** — WAF elsewhere bills normally.
7. **Keep attack evidence** — Shield attack IDs, timeframes, and affected ARNs are needed to substantiate a credit request.
8. **Credits are for covered services** — verify a given service/spike qualifies before assuming reimbursement.
9. **Unsubscribing before the term** — understand the commitment terms; early changes may not stop billing.
10. **Cost of health checks** — Route 53 health checks (recommended for detection) add small separate costs.
11. **Multi-account cost attribution** — data transfer and WAF (included) costs land in the resource-owning accounts; the subscription lands at the payer.
12. **Not a cost-control tool for normal traffic** — Shield reduces attack risk/cost, but legitimate traffic costs are your responsibility.

---

## Troubleshooting / FAQ

| Question / Issue                            | Answer / Fix                                                |
| ------------------------------------------- | ----------------------------------------------------------- |
| Charged the fee in multiple accounts?       | Fee is per consolidated-billing family; verify billing setup |
| Bill spiked during an attack                | Open a DDoS cost-protection case with Shield attack evidence |
| Which spikes are covered?                   | Attack-driven spikes on protected covered resources         |
| Can I cancel mid-term?                       | Review the 1-year commitment terms; billing may continue    |
| WAF charges appeared                        | WAF is free only on protected resources; check the resource |
| Unexpected DTO charges                      | DTO on protected resources is separate from the subscription |

---

## Best Practices

1. **Subscribe once at the org level** and cover all member accounts.
2. **Protect the resources that matter** so cost protection can apply to them.
3. **Keep attack evidence** (Shield attack IDs, timeframes) for credit requests.
4. **Model the cost/benefit** — the fee is justified for orgs with real DDoS exposure and multiple protected resources (WAF included offsets some).
5. **Open cost-protection cases promptly** after covered attacks.
6. **Account for DTO and health-check costs** separately in budgeting.
7. **Attribute costs** clearly across payer vs member accounts.

---

## Useful Links

- [Shield pricing & DDoS cost protection](https://aws.amazon.com/shield/pricing/)
- [Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
- [Requesting cost protection credits](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-responding.html)
- [Consolidated billing](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html)

---
