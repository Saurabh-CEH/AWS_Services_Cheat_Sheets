# AWS VPC - Network Analysis & Visibility Tools Cheat Sheet

## Overview

Three VPC tools help you **see and reason about** network traffic and reachability without touching production packets (except Traffic Mirroring, which copies them):

| Tool                       | What it does                                                    | Data plane? |
| -------------------------- | --------------------------------------------------------------- | ----------- |
| **Traffic Mirroring**      | Copies real packets from an ENI to a monitoring target          | **Yes** (copies packets) |
| **Reachability Analyzer**  | Checks if a path exists between a source and destination        | No (config model) |
| **Network Access Analyzer**| Finds all paths matching security intent across the network     | No (automated reasoning) |

**Key point:** Reachability Analyzer and Network Access Analyzer are **static configuration analysis** (no packets sent) — they model SGs/NACLs/route tables. Traffic Mirroring is the packet-capture option when you need real payloads.

---

## 1. Traffic Mirroring

Copies network traffic from a source ENI and sends the copy (VXLAN-encapsulated) to a target for deep packet inspection, IDS, or troubleshooting — the originals continue normally.

| Component        | Description                                                         |
| ---------------- | ------------------------------------------------------------------- |
| **Source**       | The ENI whose traffic is copied (or requester-managed ENIs from RDS/ElastiCache) |
| **Target**       | Where copies go — an ENI, **NLB (UDP 4789)**, or **GWLB endpoint**  |
| **Filter**       | Rules defining which packets to mirror (protocol, ports, CIDR, direction) |
| **Session**      | Links source + target + filter (with a session number for priority) |

**Requirements & caveats:**
- Source and target must be in the **same VPC or connected** (peering/TGW).
- Target must **allow UDP 4789**; an NLB target needs a **UDP 4789 listener** (removing it silently breaks mirroring).
- Source must have a **route** to the target; target SG/NACL must not drop mirrored traffic.
- **Nitro** instances are the general source; a specific list of **non-Nitro** types is also supported (C4, D2, G3, G3s, H1, I3, M4, P2, P3, R4, X1, X1e) — **T2 is not**, T3 is.
- **Packets are truncated to MTU** when the target is a standalone instance and the encapsulated packet exceeds its MTU — set the source MTU **54 bytes** (IPv4) / **74 bytes** (IPv6) below the target MTU.
- **Up to 100 sources per target** on certain instance types; 10 on others.
- **Cross-account (shared VPC):** participants manage only their own sessions/targets, not the owner's, and vice versa.
- Place the target in the **same AZ** as the source to reduce latency/cost.

```bash
aws ec2 create-traffic-mirror-target --network-load-balancer-arn arn:aws:elasticloadbalancing:...
aws ec2 create-traffic-mirror-filter --description "capture tcp"
aws ec2 create-traffic-mirror-session \
  --network-interface-id eni-0src --traffic-mirror-target-id tmt-0abc \
  --traffic-mirror-filter-id tmf-0abc --session-number 1 --virtual-network-id 42
```

> **Common use:** capturing packets on NLB/ALB/target ENIs you can't SSH into (TCP-reset diagnosis, IDS).

---

## 2. Reachability Analyzer

A **static configuration analysis** tool that checks whether a network path exists between a **source** and a **destination** (ENI, instance, IGW, TGW, etc.) — by **modeling** SGs, NACLs, route tables, peering, etc. It does **not** send packets.

- **Reachable** → produces **hop-by-hop** details of the virtual path.
- **Not reachable** → identifies the **blocking component** and an **explanation code**.
- You define a **path** (source → destination, optional protocol/port); analyze it **on demand** any time config changes.
- Source and destination must be in the **same Region** (cross-Region path analysis isn't supported directly).
- Great for "why can't A reach B?" — it points at the exact SG/NACL/route/missing-gateway.

```bash
aws ec2 create-network-insights-path \
  --source eni-0src --destination eni-0dst --protocol tcp --destination-port 443
aws ec2 start-network-insights-analysis --network-insights-path-id nip-0abc
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-0abc
```

> Also useful proactively: re-run a saved path to confirm intended connectivity still holds after changes.

---

## 3. Network Access Analyzer

Uses **automated reasoning** to find **all** network paths that match your **security intent**, expressed as a **Network Access Scope**. It produces **findings** for paths matching your conditions — for audit/compliance (e.g., "is anything publicly reachable that shouldn't be?").

| Concept                | Description                                                        |
| ---------------------- | ------------------------------------------------------------------ |
| **Network Access Scope** | JSON defining what paths you care about; at least one match condition |
| **MatchPaths**         | Path types to **identify** (source/destination via resource or packet-header statements) |
| **ExcludePaths**       | Path types to **exclude** from findings                            |
| **Finding**            | A path that matches ≥1 match condition and **no** exclude condition |

- Ships with **Amazon-managed scopes** (e.g., internet-ingress/egress) and supports custom scopes.
- Example uses: find publicly accessible resources; verify two subnets that should be **isolated** truly are.
- Static analysis — no packets sent; reasons over the config across accounts/Region.

```bash
aws ec2 create-network-insights-access-scope --match-paths file://scope.json
aws ec2 start-network-insights-access-scope-analysis --network-insights-access-scope-id nis-0abc
aws ec2 get-network-insights-access-scope-analysis-findings --network-insights-access-scope-analysis-id nisa-0abc
```

---

## Which Tool When?

```
Need the actual packets / payloads (IDS, PCAP)?      → Traffic Mirroring
"Why can't A reach B?" (single path debug)           → Reachability Analyzer
"What is (or isn't) reachable across my network?"    → Network Access Analyzer
   (audit intent: public exposure, isolation)
```

| Question                                   | Tool                       |
| ------------------------------------------ | -------------------------- |
| Capture real traffic for inspection        | Traffic Mirroring          |
| Debug one source→destination path          | Reachability Analyzer      |
| Audit exposure/isolation across many paths | Network Access Analyzer    |

---

## Gotchas & Caveats

1. **Reachability & Network Access Analyzer don't test the data plane** — a "reachable" result means config allows it, not that the app is actually up.
2. **Reachability Analyzer is same-Region** for a given path; plan cross-Region checks per-Region.
3. **Traffic Mirroring needs UDP 4789 open** and (for NLB targets) a UDP 4789 listener — silent failure otherwise.
4. **Traffic Mirroring truncates to MTU** for standalone-instance targets — size the source MTU below the target's.
5. **Traffic Mirroring instance-type support** varies (Nitro + a specific non-Nitro list; **not T2**).
6. **Network Access Analyzer findings = matched intent**, tune MatchPaths/ExcludePaths to avoid noise.
7. **These are analysis/visibility tools, not enforcement** — they tell you what's possible; you still fix SGs/NACLs/routes (or use VPC BPA / Network Firewall to enforce).
8. **Costs**: Traffic Mirroring adds data-processing/target costs; the analyzers charge per analysis/finding — check pricing.

---

## Useful Links

- [Traffic Mirroring — how it works](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-how-it-works.html)
- [Traffic Mirroring limitations](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-network-limitations.html)
- [What is Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/what-is-reachability-analyzer.html)
- [How Reachability Analyzer works](https://docs.aws.amazon.com/vpc/latest/reachability/how-reachability-analyzer-works.html)
- [What is Network Access Analyzer](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/what-is-network-access-analyzer.html)
- [Network Access Scopes / match conditions](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/match-paths.html)

---

*Sources: AWS official documentation. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
