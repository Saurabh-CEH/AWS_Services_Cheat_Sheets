# AWS Network Firewall - Stateless Rule Groups Cheat Sheet

## Overview

**Stateless rule groups** are the first, fast filtering layer in a Network Firewall policy. They match on the **5-tuple** (source/dest IP, source/dest port, protocol) without connection tracking, and decide to **pass**, **drop**, or **forward to the stateful engine**.

**Key point:** Stateless rules run **before** stateful rules and are evaluated by **priority**. To get deep inspection, the matching (or default) action must be **`aws:forward_to_sfe`** — otherwise the stateful rules never see the traffic.

---

## Core Concepts

| Element                    | Description                                                     |
| -------------------------- | --------------------------------------------------------------- |
| **Stateless rule group**   | Ordered (by priority) set of 5-tuple rules                      |
| **Rule priority**          | Lower number evaluated first; first match wins                  |
| **Match criteria**         | Source/dest CIDR, source/dest port ranges, protocol, TCP flags  |
| **Actions**                | `aws:pass`, `aws:drop`, `aws:forward_to_sfe`, custom actions    |
| **Capacity**               | Reserved at creation; can't be increased in place               |
| **Custom actions**         | Publish CloudWatch metrics / set dimensions on match            |

---

## Stateless Actions

| Action                 | Behavior                                                        |
| ---------------------- | --------------------------------------------------------------- |
| **`aws:pass`**         | Allow and stop evaluation (does NOT go to stateful engine)      |
| **`aws:drop`**         | Drop the packet, stop evaluation                                |
| **`aws:forward_to_sfe`** | Send to the **stateful** engine for deep inspection           |
| **Custom action**      | Emit CloudWatch metrics (named dimensions) + a standard action  |

> `aws:pass` at the stateless layer **bypasses stateful inspection** — use `aws:forward_to_sfe` when you want the stateful rules to evaluate the traffic.

---

## Rule Evaluation

```
Packet → Stateless rule group(s), evaluated by PRIORITY (low → high)
        |
        ├── match aws:pass          → allowed, NO stateful inspection
        ├── match aws:drop          → dropped
        ├── match aws:forward_to_sfe→ hand off to stateful engine
        └── no match                → STATELESS DEFAULT ACTION applies
```

- **Stateless default actions** (set in the firewall policy) decide unmatched packets — typically `aws:forward_to_sfe` so everything gets stateful inspection.
- Separate **fragment default action** handles fragmented packets.

---

## Match Criteria (5-tuple + flags)

| Field              | Notes                                             |
| ------------------ | ------------------------------------------------- |
| Source/dest CIDR   | IPv4/IPv6 ranges                                  |
| Source/dest ports  | Port ranges (e.g., 443, 1024–65535)               |
| Protocol           | By IANA number (6=TCP, 17=UDP, 1=ICMP, etc.)      |
| TCP flags          | Match specific flag combinations (SYN, ACK, ...)  |

---

## CLI

```bash
# Create a stateless rule group (reserve capacity generously)
aws network-firewall create-rule-group \
  --rule-group-name allow-web-fwd --type STATELESS --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "StatelessRulesAndCustomActions": {
        "StatelessRules": [
          {
            "Priority": 10,
            "RuleDefinition": {
              "MatchAttributes": {
                "Sources": [{"AddressDefinition":"10.0.0.0/16"}],
                "DestinationPorts": [{"FromPort":443,"ToPort":443}],
                "Protocols": [6]
              },
              "Actions": ["aws:forward_to_sfe"]
            }
          }
        ]
      }
    }
  }'

# Inspect
aws network-firewall describe-rule-group --rule-group-name allow-web-fwd --type STATELESS
```

---

## Pricing

| Item                     | Cost                                          |
| ------------------------ | --------------------------------------------- |
| Rule groups              | No separate charge for the rule group object  |
| Firewall endpoint + data | Hourly endpoint + per-GB processed (policy-wide) |

---

## Quotas (defaults, adjustable)

| Resource                                | Default limit |
| --------------------------------------- | ------------- |
| Rule group capacity                     | Reserved at creation (fixed) |
| Stateless rule groups per policy        | (adjustable)  |
| Rules per stateless rule group          | Bounded by capacity |

---

## Gotchas & Caveats

1. **Capacity is reserved at creation and can't be increased** — you must recreate the rule group at a higher capacity. Estimate generously up front.
2. **`aws:pass` skips the stateful engine** — if you want deep inspection, use `aws:forward_to_sfe`, not `aws:pass`.
3. **Stateless = no connection tracking** — return traffic isn't auto-allowed; you must account for both directions (or forward to the stateful engine, which is stateful).
4. **Priority order matters** — lowest number first, first match wins; a broad low-priority rule can shadow specific ones.
5. **The stateless default action is policy-level** — set it to `aws:forward_to_sfe` so unmatched traffic reaches stateful rules; a default of `aws:drop` silently blocks everything unmatched.
6. **Fragments have a separate default action** — forgetting it can drop or pass fragments unexpectedly.
7. **Protocol is numeric** — use IANA numbers (6/17/1), not names.
8. **Custom actions only add metrics/dimensions** — they still pair with a standard action; they don't create new forwarding behavior.
9. **Stateless rules can't do domain/Layer-7 matching** — that's the stateful engine's job.
10. **Capacity accounting is per-rule-complexity**, not just rule count — complex match attributes consume more capacity.
11. **Changes propagate but aren't instant** — allow time for policy updates to take effect on endpoints.
12. **Deleting a rule group referenced by a policy fails** — remove the reference first.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Stateful rules never match                  | Stateless action/default is `pass` or `drop`    | Use `aws:forward_to_sfe` for traffic that needs inspection |
| Return traffic dropped                      | Stateless has no state                          | Forward to stateful engine, or allow both directions       |
| Unmatched traffic blocked                   | Stateless default = `aws:drop`                  | Set default to `aws:forward_to_sfe` (or pass) as intended  |
| Can't grow rule group                       | Capacity fixed at creation                      | Recreate with higher capacity                              |
| Fragments behaving oddly                    | Fragment default action not set                 | Configure the fragment default action                     |
| Specific rule never hit                     | Shadowed by lower-priority broad rule           | Reorder priorities                                         |

---

## Best Practices

1. **Default stateless action = `aws:forward_to_sfe`** so all traffic gets stateful inspection unless explicitly fast-pathed.
2. **Use stateless rules only for cheap, obvious pass/drop** (e.g., drop known-bad ports) and forward the rest.
3. **Reserve generous capacity** at creation.
4. **Order priorities carefully** — specific before broad.
5. **Set the fragment default action** deliberately.
6. **Add custom actions for metrics** on important match categories.
7. **Keep Layer-7/domain logic in the stateful layer.**

---

## Useful Links

- [Stateless rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateless-rule-groups.html)
- [Rule group capacity](https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-group-capacity.html)
- [Stateless actions](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateless-rule-action.html)
- [Custom actions & metrics](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateless-custom-actions.html)
- [Rule group quotas](https://docs.aws.amazon.com/network-firewall/latest/developerguide/quotas.html)

---
