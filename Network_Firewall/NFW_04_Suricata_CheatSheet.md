# AWS Network Firewall - Suricata Rules Cheat Sheet

## Overview

AWS Network Firewall's stateful engine is **Suricata-compatible** — all stateful rule groups (5-tuple, domain lists, and raw Suricata) are compiled to Suricata rules internally. Writing raw Suricata gives you the full signature language: protocol keywords, content matching, flow state, app-layer parsing, and metadata.

**Key point:** Network Firewall supports the **libpcre / Suricata rule syntax with some exceptions**. Not every Suricata keyword or feature is supported — check the compatibility reference before relying on an advanced keyword.

---

## Rule Anatomy

```
action  protocol  src_ip src_port  direction  dst_ip dst_port  (options; ... sid:NNN; rev:N;)
```

Example:
```
drop tls $HOME_NET any -> $EXTERNAL_NET any (tls.sni; content:"malware.example.com"; msg:"block C2"; sid:1000001; rev:1;)
```

| Part            | Meaning                                                             |
| --------------- | ------------------------------------------------------------------- |
| **action**      | `pass` / `drop` / `reject` / `alert`                                |
| **protocol**    | `ip`, `tcp`, `udp`, `icmp`, or app-layer: `http`, `tls`, `dns`, `ssh`, etc. |
| **src/dst**     | IPs/vars (`$HOME_NET`, `$EXTERNAL_NET`), CIDRs, `any`               |
| **ports**       | numbers, ranges (`[80,443]`, `1024:`), `any`                        |
| **direction**   | `->` (one way) or `<>` (bidirectional)                              |
| **options**     | keywords in `()`, semicolon-separated                               |
| **`sid`**       | unique rule ID (**required**)                                       |
| **`rev`**       | rule revision number                                                |

---

## Actions (recap)

| Action   | Behavior                                                    |
| -------- | ----------------------------------------------------------- |
| `pass`   | Allow; stops scanning the rest of that flow                 |
| `drop`   | Silently drop                                               |
| `reject` | Drop + send TCP RST (TCP only)                              |
| `alert`  | Log to ALERT logs, keep evaluating                          |

---

## Rule Variables

Defined per rule group (or policy) — reference them in rules.

| Variable         | Typical meaning                                   |
| ---------------- | ------------------------------------------------- |
| `$HOME_NET`      | Your protected/internal CIDRs                     |
| `$EXTERNAL_NET`  | Everything else (often `!$HOME_NET`)              |
| Custom `$VARS`   | Port sets, IP sets you define                     |

> Wrong `$HOME_NET`/`$EXTERNAL_NET` scoping is a top cause of rules over- or under-matching.

---

## Common Keywords

### Flow / state
| Keyword                     | Use                                                         |
| --------------------------- | ----------------------------------------------------------- |
| `flow:established`          | Match only established connections                          |
| `flow:established,to_server`| Client→server on established flow                           |
| `flow:established,to_client`| Server→client on established flow                           |
| `flow:stateless`            | Match without state tracking                                |

> Use the `flow` keyword to avoid matching lower-layer packets before the app protocol is identified (e.g., a TCP rule matching the first handshake packet before TLS/HTTP is known).

### Content matching
| Keyword               | Use                                             |
| --------------------- | ----------------------------------------------- |
| `content:"..."`       | Byte/string match                               |
| `nocase`              | Case-insensitive (follows a content match)      |
| `pcre:"/regex/"`      | Regex match                                     |
| `dsize`               | Payload size condition                          |
| `offset` / `depth`    | Where in the payload to look                    |

### Application-layer
| Keyword               | Use                                             |
| --------------------- | ----------------------------------------------- |
| `tls.sni`             | Match on TLS Server Name Indication             |
| `http.host`           | Match on HTTP Host header                       |
| `http.uri` / `http.method` / `http.user_agent` | HTTP fields              |
| `dns.query`           | Match on DNS query name                         |
| `ja3.hash` / `ja3s.hash` | TLS fingerprints                             |

### Metadata
| Keyword    | Use                                     |
| ---------- | --------------------------------------- |
| `msg:"..."`| Human-readable message (appears in logs)|
| `sid:N`    | Unique rule ID (required)               |
| `rev:N`    | Revision                                |
| `priority:N` | Order within an action group (default action order only) |
| `metadata` | Free-form tags                          |

---

## Example Rules

```
# Block a domain over TLS by SNI
drop tls $HOME_NET any -> $EXTERNAL_NET any (tls.sni; content:"c2.example.com"; nocase; msg:"block C2 SNI"; sid:1000001; rev:1;)

# Alert on a scanner user-agent over HTTP
alert http any any -> any any (http.user_agent; content:"sqlmap"; nocase; msg:"scanner UA"; sid:1000002; rev:1;)

# Allow established outbound HTTPS from home net
pass tls $HOME_NET any -> $EXTERNAL_NET 443 (flow:established,to_server; msg:"allow outbound https"; sid:1000003; rev:1;)

# Block a specific DNS query name
drop dns $HOME_NET any -> any any (dns.query; content:"badsite.example.net"; nocase; msg:"block dns"; sid:1000004; rev:1;)

# Allow TCP keep-alives to work with Application drop established (bidirectional)
pass tcp any any -> any any (msg:"keep-alives to_server"; tcp.flags:A; dsize:0; flow:established,to_server; sid:1000010; rev:1;)
pass tcp any any -> any any (msg:"keep-alives to_client"; tcp.flags:A; dsize:0; flow:established,to_client; sid:1000011; rev:1;)
```

---

## CLI

```bash
# Raw Suricata rules string
aws network-firewall create-rule-group \
  --rule-group-name suricata-sigs --type STATEFUL --capacity 500 \
  --rule-group '{"RulesSource":{"RulesString":"drop tls $HOME_NET any -> $EXTERNAL_NET any (tls.sni; content:\"c2.example.com\"; nocase; sid:1000001; rev:1;)"}}'

# Set rule variables on the rule group (HOME_NET etc.) via the RuleVariables field
```

---

## Gotchas & Caveats

1. **`sid` is required and must be unique** within your rules — duplicate/missing `sid` fails the rule-group update.
2. **Not all Suricata keywords are supported** — Network Firewall implements Suricata-compatible syntax **with exceptions**; validate against the compatibility reference.
3. **A syntax error fails the whole rule-group create/update** — one bad rule blocks the batch.
4. **`pass` stops scanning the rest of the flow** — a broad `pass` can unintentionally allow subsequent packets you meant to inspect.
5. **Protocol-layer ordering** — a `tcp` rule may match the first handshake packet before the app protocol (TLS/HTTP) is identified; use `flow` and app-layer keywords to avoid premature matches.
6. **`tls.sni` / `http.host` are cleartext-dependent** — encrypted SNI / direct-IP access evades them unless TLS inspection is on.
7. **`priority` only applies in default (action) order** — in strict order, order is by rule-group priority then rule order.
8. **Rule variables must match your address space** — wrong `$HOME_NET`/`$EXTERNAL_NET` causes over/under-matching.
9. **Capacity is consumed by rule complexity**, not just count — heavy `pcre`/content rules cost more; capacity is fixed at creation.
10. **`reject` is TCP-only** — for UDP/other, `reject` behaves like `drop`.
11. **Managed rule groups are Suricata rules too** — they occupy capacity and evaluate in the same engine; validate interactions.
12. **App-layer rules can span multiple packets** — pair with the right stateful default action (see the Stateful Rules sheet) so a default drop doesn't fire before SNI/Host is seen.

---

## Best Practices

1. **Always set a unique `sid`** and use `rev` for versioning.
2. **Use `flow` + app-layer keywords** to match precisely and avoid premature protocol matches.
3. **Deploy new signatures as `alert` first**, review ALERT logs, then switch to `drop`/`reject`.
4. **Set rule variables** (`$HOME_NET`, `$EXTERNAL_NET`) to your real CIDRs.
5. **Keep `pcre`/content rules lean** to control capacity and latency.
6. **Validate syntax** before deploying; one bad rule blocks the group.
7. **Prefer app-layer keywords** (`tls.sni`, `http.host`) over raw `content` where possible.
8. **Reserve generous capacity** at creation — you can't grow it in place.

---

## Useful Links

- [Suricata compatibility in Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-compatibility.html)
- [Working with Suricata rules](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-suricata.html)
- [Rule evaluation order](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-rule-evaluation-order.html)
- [Suricata rules examples](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-examples.html)
- [Suricata User Guide (rules)](https://docs.suricata.io/en/latest/rules/index.html)

---
