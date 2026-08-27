# AWS Network Firewall - Stateful Rules & Default Actions Cheat Sheet

## Overview

**Stateful rule groups** are the deep-inspection layer of Network Firewall. They are **connection-aware**, supporting 5-tuple rules, **domain allow/deny lists** (HTTP Host / TLS SNI), raw **Suricata** signatures, and **AWS managed threat rule groups**. This sheet focuses on rule **types**, **evaluation order**, and choosing the right **default actions**.

**Key point:** Stateful evaluation uses either **default (action) order** or **strict order**. Strict order is predictable, sequential, and generally recommended; default order groups rules by action and can surprise you.

> For raw Suricata rule syntax, keywords, and examples, see the dedicated **Suricata** cheat sheet (`NFW_Suricata_CheatSheet.md`).

---

## Stateful Rule Types

| Type                        | Description                                                 |
| --------------------------- | ----------------------------------------------------------- |
| **5-tuple (standard)**      | Suricata-style IP/port/protocol rules with actions          |
| **Domain list**             | Allow/deny by domain via HTTP `Host` header and TLS **SNI** |
| **Suricata compatible**     | Raw Suricata rule strings — full signature capability       |
| **AWS managed rule groups** | Curated threat intel (botnet, malware, abused domains, etc.) |

---

## Stateful Actions

| Action     | Behavior                                                       |
| ---------- | -------------------------------------------------------------- |
| **pass**   | Allow the packet/flow, stop evaluating                         |
| **drop**   | Silently drop                                                  |
| **reject** | Drop and send a TCP reset (RST) to the sender                  |
| **alert**  | Log a match (to ALERT logs) without blocking; keep evaluating  |

---

## Rule Evaluation Order

| Mode                       | Behavior                                                              |
| -------------------------- | -------------------------------------------------------------------- |
| **Default (action) order** | Rules evaluated by **action precedence: pass → drop → reject → alert**; within an action group, the Suricata `priority` keyword (1–65535, lowest first) sets order. A `pass` match stops scanning the rest of that flow. |
| **Strict order**           | Rules evaluated in the exact order you define (by rule-group priority, then rule order); you set explicit **default actions** (see below) |

```
Strict order:
  Rule 1 (priority 1) → Rule 2 → Rule 3 → ... → default actions
  First terminating match wins (like a traditional firewall)
```

> Choose the order mode when creating the **firewall policy**. Switching later can change effective behavior — validate in `alert` mode.

### Strict-Order Default Actions

With strict order you choose the **default action(s)** the engine applies when no rule matches. Beyond the basic `Drop all` / `Drop established`, AWS added **application-layer** default actions with enhanced handling of segmented (multi-packet) traffic.

**Drop actions — choose `none` or exactly one:**

| Default drop action                              | Behavior                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------ |
| **Drop all**                                     | Drops all packets not matched by a pass rule                             |
| **Drop established**                             | Drops packets in established client→server connections; allows L3/4 handshake packets. For **connectionless** protocols (UDP/ICMP) it drops all — write explicit pass rules |
| **Application drop established (bidirectional)**  | Drops server-initiated banner packets + established-connection packets; waits for **TLS SNI / HTTP Host** on segmented traffic before applying rules |
| **Application drop established (server-directed only)** | Drops only client→server established TCP/IP packets that match no pass rule; **preserves** server→client traffic (banners, RST, keep-alives) |

**Alert actions — choose `none`, one, or `Alert all` plus others:**

| Default alert action                             | Behavior                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------ |
| **Alert all**                                    | Logs an ALERT on every packet (shows what `Drop all` would drop)         |
| **Alert established**                            | Logs ALERT on established-connection packets (preview of `Drop established`) |
| **Application alert established (bidirectional)** | ALERT preview of the bidirectional application-drop behavior             |
| **Application alert established (server-directed only)** | ALERT preview of the server-directed application-drop behavior     |

> **Domain-list rule groups in strict order require `Drop established`** (or an application-drop-established default) — Network Firewall needs an **established connection** to evaluate pass/drop for domain lists. Without a drop-established default, your domain allow/deny logic won't enforce as expected.

> **Application drop established (bidirectional) can drop TCP flow-control packets** (window updates, keep-alives, resets) seen right after the handshake before a pass rule applies. If that breaks apps, add explicit pass rules (e.g., `pass tcp ... tcp.flags:A; dsize:0; window:!0; flow:established,to_client; ...`) — or use the **server-directed only** variant, which preserves server→client control packets.

**Valid-combination caveats:**
- You can choose **at most one** drop action.
- If **Drop established** or **Alert established** is selected, you **cannot** also select any of the Application drop/alert established (bidirectional or server-directed) actions, and vice versa.
- For rules that match **application-layer data spanning multiple packets** (e.g., HTTP headers), a plain default drop can trigger too early — prefer an **application** drop-established default or don't use a default drop and write app-layer-specific drop rules instead.

### When to Use Which Default Action

| Your goal / situation                                              | Recommended default action(s)                          |
| ------------------------------------------------------------------ | ------------------------------------------------------- |
| **Just observe / tune before enforcing**                           | `Alert all` (or `Alert established`) — nothing is dropped, you see what would be |
| **Default-deny, allowlist model, TCP + non-TCP**                   | `Drop all` + explicit `pass` rules for everything allowed (remember: for UDP/ICMP, `Drop established` drops all — `Drop all` is clearer) |
| **Default-deny but keep lower-layer handshakes working**           | `Drop established` — allows L3/4 handshake, drops established client→server not matched by pass rules |
| **Using domain-list rule groups in strict order**                  | **`Drop established`** (or an Application drop-established) — required; the engine needs an established connection to evaluate domain lists |
| **App-layer (HTTP/TLS) allowlist where rules match SNI/Host across multiple packets** | `Application drop established (bidirectional)` — waits for TLS SNI / HTTP Host before deciding |
| **Server-initiated protocols (FTP/SMTP/SSH banners) or TCP control packets getting dropped** | `Application drop established (server-directed only)` — preserves server→client traffic and control packets |
| **Preview the impact of an application-drop default before enforcing** | matching `Application alert established (...)` variant  |
| **Not sure / general internet-egress filtering by domain**         | `Drop established` + domain allowlist, validated first with `Alert established` |

**Decision flow:**

```
Testing / first rollout?            → Alert all (or Alert established)
Need L3/4 handshakes to work?       → Drop established (not Drop all)
Filtering by domain (SNI/Host)?     → Drop established (required) — or Application drop established if rules span packets
Server-initiated / banner traffic breaking? → Application drop established (server-directed only)
Simple strict allowlist, all protocols?     → Drop all + explicit pass rules
```

> **Rule of thumb:** start with an **alert** default, confirm behavior in logs, then move to the matching **drop** default. Use the **Application** variants only when your pass/deny decisions depend on application-layer fields (SNI/Host) that arrive across multiple packets.

---

## Domain Filtering

| Aspect              | Detail                                                          |
| ------------------- | --------------------------------------------------------------- |
| Inspects            | HTTP `Host` header and **TLS SNI** (clear-text server name)     |
| Modes               | **Allowlist** (deny all but listed) or **Denylist** (block listed) |
| Wildcards           | `.example.com` matches subdomains                               |
| Limitation          | Encrypted SNI / IP-only access can bypass unless TLS inspection is enabled |

```json
{
  "RulesSource": {
    "RulesSourceList": {
      "TargetTypes": ["TLS_SNI", "HTTP_HOST"],
      "Targets": ["badsite.example.com", ".ads.example.net"],
      "GeneratedRulesType": "DENYLIST"
    }
  }
}
```

---

## Suricata Rules

Raw Suricata signatures are one of the stateful rule-group types. Full syntax, keywords (`flow`, `tls.sni`, `http.host`, `content`, `pcre`, `sid`/`rev`), rule variables, and examples are in the dedicated **Suricata** cheat sheet (`NFW_Suricata_CheatSheet.md`).

Quick example:
```
drop tls $HOME_NET any -> $EXTERNAL_NET any (tls.sni; content:"malware.example.com"; nocase; msg:"block C2"; sid:1000001; rev:1;)
```

---

## CLI

```bash
# Domain denylist stateful rule group (strict-order compatible)
aws network-firewall create-rule-group \
  --rule-group-name deny-domains --type STATEFUL --capacity 100 \
  --rule-group '{"RulesSource":{"RulesSourceList":{"TargetTypes":["TLS_SNI","HTTP_HOST"],"Targets":["badsite.example.com"],"GeneratedRulesType":"DENYLIST"}}}'

# Raw Suricata rules
aws network-firewall create-rule-group \
  --rule-group-name suricata-sigs --type STATEFUL --capacity 500 \
  --rule-group '{"RulesSource":{"RulesString":"drop tls $HOME_NET any -> $EXTERNAL_NET any (tls.sni; content:\"c2.example.com\"; sid:1000001; rev:1;)"}}'

# Add an AWS managed stateful rule group by ARN in the policy
# (reference arn:aws:network-firewall:...:aws-managed/stateful-rulegroup/...)
```

---

## Pricing

| Item                     | Cost                                          |
| ------------------------ | --------------------------------------------- |
| Rule groups              | No per-object charge                          |
| Endpoint + data          | Hourly endpoint + per-GB processed (policy-wide) |

---

## Gotchas & Caveats

1. **Default vs strict order changes precedence** — strict order is sequential/predictable; default (action) order applies engine precedence that can surprise. Pick deliberately at policy creation.
2. **Domain filtering relies on cleartext SNI/Host** — it does **not** decrypt TLS by default; encrypted-SNI/ESNI or direct-IP access evades domain rules unless **TLS inspection** is enabled.
3. **Capacity reserved at creation, not resizable** — recreate the rule group to grow it; estimate high.
4. **Suricata syntax errors fail rule-group create/update** — validate rules; a bad `sid`/keyword blocks the whole update.
5. **`alert` doesn't block** — it only logs; deploy in alert first, then switch to drop/reject.
6. **`reject` sends TCP RST** (TCP only) — useful for faster client failure, but reveals the firewall; `drop` is silent.
7. **Asymmetric routing breaks statefulness** — return traffic must traverse the same firewall/AZ (appliance mode in TGW designs).
8. **Managed rule groups update over time** — signatures change; test before enforcing drop.
9. **Strict order needs explicit default actions** (e.g., `aws:drop_established`) — omitting them changes behavior.
10. **Rule variables (`$HOME_NET`) must be set correctly** — wrong scoping makes rules over/under-match.
11. **Flow logs vs alert logs** — you only see matches in ALERT logs for `alert`/`drop`; passing traffic won't appear there.
12. **Domain lists and 5-tuple/Suricata rules in the same group** have compatibility constraints — some combinations require separate groups.
13. **Domain-list rule groups (strict order) require a `Drop established`-type default action** — without it, domain allow/deny won't enforce because the engine needs an established connection to decide.
14. **App-layer default drop can drop TCP control packets** — `Application drop established (bidirectional)` may drop window updates / keep-alives / resets right after the handshake; add explicit pass rules or use the **server-directed only** variant.
15. **Default drop can trigger too early on multi-packet app data** — for rules matching HTTP headers/etc. that span packets, prefer an **application** drop-established default (which waits for SNI/Host) or write app-layer-specific drop rules instead of a plain `Drop all`/`Drop established`.
16. **Drop/alert action combos are constrained** — at most one drop action; `Drop/Alert established` can't be mixed with the Application drop/alert established actions.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Domain block not working                    | Encrypted SNI / IP access / no TLS inspection   | Add IP rules; enable TLS inspection                        |
| Rules match unexpectedly / wrong precedence | Default (action) order semantics                | Use strict order with explicit defaults                    |
| Rule group won't update                     | Suricata syntax error                           | Validate rule syntax; fix `sid`/keywords                   |
| Nothing is blocked                          | Rules in `alert` only                           | Switch to `drop`/`reject` after validation                 |
| Return traffic dropped                      | Asymmetric routing                              | Same-AZ endpoints; appliance mode on TGW                   |
| Rules over/under-matching                   | Wrong `$HOME_NET`/`$EXTERNAL_NET`               | Fix rule variables                                         |
| Can't grow rule group                       | Capacity fixed                                  | Recreate with higher capacity                              |

---

## Best Practices

1. **Use strict rule order** for predictable, auditable policy with explicit default actions.
2. **Deploy in `alert` first**, review ALERT logs, then move to `drop`/`reject`.
3. **Layer AWS managed rule groups** (threat intel) + your domain lists + custom Suricata.
4. **Enable TLS inspection** where real domain/content enforcement is required.
5. **Reserve generous capacity** and validate Suricata syntax before deploying.
6. **Set rule variables** (`$HOME_NET`) to match your address space.
7. **Ensure symmetric routing** (appliance mode) so statefulness holds.

---

## Useful Links

- [Stateful rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups.html)
- [Stateful rule evaluation order](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-rule-evaluation-order.html)
- [Domain list rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-domain-names.html)
- [Suricata compatibility](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-compatibility.html)
- [AWS managed rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/aws-managed-rule-groups.html)
- [Rule actions](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-action.html)

---
