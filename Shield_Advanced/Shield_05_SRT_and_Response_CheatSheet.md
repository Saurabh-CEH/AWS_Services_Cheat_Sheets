# AWS Shield Advanced - SRT & Incident Response Cheat Sheet

## Overview

The **Shield Response Team (SRT)** is a group of AWS DDoS experts available to Shield Advanced customers to help **during and after** attacks. You can engage them reactively (open a case) or set up **proactive engagement**, where the SRT contacts you when health checks indicate a likely attack.

**Key point:** Proactive engagement requires **configured emergency contacts** and **associated Route 53 health checks** — set these up *before* an incident, not during one.

---

## Engagement Modes

| Mode                     | How it works                                                  |
| ------------------------ | ------------------------------------------------------------- |
| **Reactive**             | You contact the SRT (via a Support case) during an attack     |
| **Proactive engagement** | SRT proactively contacts you when health checks show impact   |
| **SRT access to WAF**    | You can grant the SRT permission to create/manage WAF rules on your behalf during an incident |

---

## Prerequisites

| Requirement                    | Detail                                                        |
| ------------------------------ | ------------------------------------------------------------- |
| Shield Advanced subscription   | Active                                                        |
| Emergency contacts             | Email + phone for the SRT to reach you                        |
| Route 53 health checks         | Associated with protections (for proactive engagement)        |
| SRT IAM role (optional)        | Grant SRT access to act on your WAF/Shield during incidents   |
| Support plan                   | Business or Enterprise Support typically required to open SRT cases |

---

## How Proactive Engagement Works

```
Route 53 health checks associated with protections
        |
        v
Shield detects anomaly + health check shows the app is degraded
        |
        v
SRT proactively CONTACTS your configured emergency contacts
        |
        v
(If SRT access granted) SRT helps apply mitigations / WAF rules
```

---

## CLI

```bash
# Set emergency contacts
aws shield associate-proactive-engagement-details \
  --emergency-contact-list \
    EmailAddress=soc@example.com,PhoneNumber=+15551234567,ContactNotes="Primary NOC" \
    EmailAddress=oncall@example.com,PhoneNumber=+15559876543

# Enable proactive engagement
aws shield enable-proactive-engagement

# Grant the SRT access to act on your account (create the role first)
aws shield associate-drt-role --role-arn arn:aws:iam::123456789012:role/AWSSRTAccessRole
# Optionally give the SRT access to specific log buckets
aws shield associate-drt-log-bucket --log-bucket aws-waf-logs-mybucket

# Disable / revoke
aws shield disable-proactive-engagement
aws shield disassociate-drt-role
```

---

## Gotchas & Caveats

1. **Set up contacts and health checks before you need them** — proactive engagement can't work without configured emergency contacts and associated Route 53 health checks.
2. **Proactive engagement depends on health-based detection** — no health checks means the SRT can't reliably know your app is impacted.
3. **SRT access to WAF requires an IAM role you create and associate** — without `associate-drt-role`, the SRT can advise but not act on your resources.
4. **Support plan matters** — opening SRT cases generally requires Business or Enterprise Support.
5. **Contact accuracy is critical** — stale emails/phone numbers mean the SRT can't reach you during an attack.
6. **SRT is not a substitute for your own runbook** — have internal escalation, WAF rules, and architecture ready.
7. **Granting SRT log-bucket access** helps them analyze, but scope it to WAF log buckets (least privilege).
8. **Revoke access appropriately** — the SRT role persists until you disassociate it; review periodically.
9. **Proactive engagement covers eligible resources** — ensure the resources you care about are protected and have health checks.
10. **Response time expectations** — the SRT helps but response isn't instantaneous; automated mitigations (Shield L3/4, WAF rate rules) act first.
11. **Multi-account** — configure engagement in the accounts that own protected resources, consistent with the org subscription.
12. **Test the workflow** — validate contacts and understand the process before a real incident.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                       |
| ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| SRT didn't proactively engage               | Proactive engagement off / no contacts / no health checks | Enable it; set contacts; associate health checks |
| SRT can't act on my WAF                     | No SRT IAM role associated                      | `associate-drt-role` with a proper role                   |
| Can't open an SRT case                      | Support plan insufficient                       | Upgrade to Business/Enterprise Support                    |
| SRT couldn't reach us                       | Stale emergency contacts                        | Update emergency contact list                             |
| SRT lacks logs to analyze                   | No log-bucket access granted                    | `associate-drt-log-bucket` (WAF log bucket)               |

---

## Best Practices

1. **Configure emergency contacts and enable proactive engagement** proactively.
2. **Associate Route 53 health checks** so proactive engagement can trigger.
3. **Create and associate the SRT IAM role** so the team can act during incidents.
4. **Grant scoped log-bucket access** to speed SRT analysis.
5. **Maintain accurate 24/7 contacts** (NOC/on-call).
6. **Keep your own runbook** (WAF rate rules, escalation) — automated defenses act first.
7. **Review SRT access and contacts periodically.**
8. **Validate the workflow** in a tabletop exercise before a real attack.

---

## Useful Links

- [Shield Response Team (SRT)](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt.html)
- [Proactive engagement](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-proactive-engagement.html)
- [Configuring SRT access](https://docs.aws.amazon.com/waf/latest/developerguide/authorize-srt.html)
- [Responding to DDoS events](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-responding.html)

---
