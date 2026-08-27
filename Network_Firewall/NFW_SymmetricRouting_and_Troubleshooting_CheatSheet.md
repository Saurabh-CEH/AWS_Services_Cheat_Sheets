# AWS Network Firewall - Symmetric Routing & Troubleshooting Cheat Sheet

## Overview

Network Firewall is a **managed, stateful** service that uses a **Gateway Load Balancer (GWLB)** to spread flows across backend firewall appliances. It **does not support asymmetric routing** — the forward (request) and return (response) traffic of a flow **must** reach the **same firewall endpoint**, or stateful features (application-layer inspection, domain lists) break and traffic is dropped.

**Key point:** Route traffic to the firewall endpoint **closest to the client in both directions**. Most "firewall is dropping / not scanning traffic" cases are routing (asymmetry or no route), not rules.

---

## Why Symmetry Matters

```
Request:  client → firewall endpoint A → server
Response: server → firewall endpoint A → client   ✅ symmetric (stateful engine sees both halves)

Response: server → firewall endpoint B → client   ❌ asymmetric → return dropped
```

- Stateful inspection tracks flow state; it must see the **3-way handshake and both directions** on the same endpoint.
- Asymmetric flows are subject to the **stream exception policy** (default: **DROP**).

---

## Keeping Routing Symmetric by Deployment Model

| Model                          | How to keep it symmetric                                                  |
| ------------------------------ | ------------------------------------------------------------------------- |
| **Centralized (inspection VPC + TGW)** | Enable **TGW appliance mode** on the inspection VPC's attachment; route forward + return via the firewall attachment. Appliance mode keeps a flow on one ENI/AZ for its life. |
| **Decentralized (per-VPC, internet ingress)** | Use an **IGW edge (gateway) route table association** to send inbound through the firewall endpoint, plus an outbound route in the app subnet. |
| **NAT gateway downstream**     | Ensure the **NAT gateway's subnet** routes traffic through the firewall endpoint. |
| **TGW-attached firewall (native)** | Associate the firewall's TGW attachment in the correct route tables so both directions traverse it. |

> **Appliance mode** = the TGW picks one network interface (by flow hash) in the appliance VPC and uses the **same** interface for return traffic, so the flow stays in one AZ for its life.

---

## Detecting Asymmetric Routing (AWS's documented test)

1. Create/associate an **empty strict-order** policy; stateless default = **forward to stateful rule groups** (full + fragments). No stateful default actions needed for the test.
2. Enable **Alert** and **Flow** logging to **separate** CloudWatch log groups.
3. Add one Suricata test rule:
   ```
   alert tcp any any -> any any (msg:"Routing is symmetric. You can safely remove this test rule."; flow:established; sid:123456;)
   ```
4. From an instance behind the firewall, `curl https://www.amazon.com`.
5. Search the **alert** log group for the SNI (`www.amazon.com`). An alert with `app_proto: tls` and `flow:established` means the engine saw the handshake **bidirectionally** → **routing is symmetric**.
6. (Optional) Copy the `flow_id` and search the **flow** log group. Seeing **both** a request (client→server) and response (server→client) netflow event confirms symmetry. Seeing **only the request** side means the **return path bypasses the firewall** (asymmetric) — or a NACL/SG is blocking inbound at the server.

Other tools: **VPC Reachability Analyzer**, the **stateless rule group analyzer**, and flow/alert logs.

---

## Stream Exception Policy (asymmetric / mid-stream packets)

When the firewall sees packets it can't associate with a tracked flow (e.g., asymmetrically-forwarded connections), the **stream exception policy** decides what happens:

| Policy      | Behavior                                                                 | Metrics affected                                         |
| ----------- | ------------------------------------------------------------------------ | -------------------------------------------------------- |
| **DROP** (default) | Drop matching packets                                              | `StreamExceptionPolicyPackets` **and** `DroppedPackets`  |
| **REJECT**  | Send TCP reset; drop further packets on the connection                   | `StreamExceptionPolicyPackets` **and** `RejectedPackets` (then `DroppedPackets`) |
| **CONTINUE**| Keep processing the packets against rules (may still be dropped by a rule) | `StreamExceptionPolicyPackets` (elevated values here are not necessarily bad) |

> A **high `StreamExceptionPolicyPackets`** during a latency/drop window is a strong signal of **asymmetric forwarding**. Also note: **unidirectional stateless pass rules** can create asymmetric forwarding when the stateless default is "forward to stateful" — write **pairs** of stateless rules (forward + return) so both directions reach the stateful engine.

---

## Flow Operations & the Firewall State Table

The **firewall state table** is where Network Firewall tracks stateful flows. A **flow** is traffic sharing the same **Source, SourcePort, Destination, DestinationPort, Protocol, and Direction**. Only flows processed by **stateful** rules are tracked; entries persist until they're flushed, terminate naturally, or **time out from inactivity**. **Flow operations** let you inspect and manage that table asynchronously:

| Operation        | What it does                                                                 |
| ---------------- | ---------------------------------------------------------------------------- |
| **Flow capture** | Collects info about **active flows** matching your filter (troubleshooting/visibility) |
| **Flow flush**   | **Removes** matching flows from the state table                              |
| **Flow filter**  | Scope of an operation — Source/Dest IP, ports, protocol. **Up to 20 filters** per operation |

**Caveats & considerations:**
- **Flush honors the stream exception policy** — flushed flows are treated per your DROP/REJECT/CONTINUE setting. Review it before flushing.
- After a flush, **subsequent matching traffic is a NEW flow** and is re-evaluated against your **current** rules.
- **Only the firewall owner** can run flow operations — a VPC endpoint association owner who doesn't own the firewall **cannot**.
- **Broad capture filters** (wide IP ranges) can hit operation limits — narrow with ports/protocol/tighter ranges.
- **Async & per-firewall** — each operation runs on **one firewall at a time**; flushes propagate and may mark flows at slightly different times. Repeat per firewall for multi-firewall setups.
- **Throttled to one concurrent request per firewall per AZ** (e.g., a 2-AZ firewall allows 2 concurrent requests, one per AZ).

**Use cases:** confirm a specific connection is actually tracked (capture), or force re-evaluation of in-flight flows after a rule change / to clear stuck or malicious connections (flush).

---

## Key CloudWatch Metrics for Triage

| Metric                        | What it tells you                                              |
| ----------------------------- | -------------------------------------------------------------- |
| `ReceivedPackets`             | If **0**, traffic isn't reaching the firewall → fix route tables (route to the endpoint) |
| `DroppedPackets`              | Packets dropped by rules / stream exception / fail-close       |
| `StreamExceptionPolicyPackets`| Asymmetric / out-of-state packets hitting the exception policy  |
| `RejectedPackets`             | Packets rejected (TCP RST)                                      |
| `PassedPackets`               | Packets passed                                                 |
| GWLB endpoint: `PacketsDropped`, `ActiveConnections`, `BytesProcessed` | PrivateLink/GWLBe-level drops (open a Support case if `PacketsDropped` rises) |

---

## GWLB Idle Timeout (350s) — long-lived connections

- Network Firewall uses a **GWLB endpoint** to distribute flows; GWLB has a **fixed 350-second idle timeout** for TCP flows.
- On idle timeout (or connection close), the flow is removed from the state table; later non-SYN packets for that 5-tuple may be **dropped**, and a new connection with the same 5-tuple may land on a **different backend** appliance.
- **Fix:** set **TCP keep-alive** on client/server **below 350s**, **or** increase the **idle timeout** in the Network Firewall firewall-policy settings **above** your app's timeout.

---

## Scaling & Fail-Close

- Each endpoint is **zonal** and scales **independently up to ~100 Gbps**; scaling is automatic (no config).
- **Scale-up** completes in minutes; **scale-down** is gradual over hours.
- **Fail-close:** sudden large spikes that briefly exceed capacity can cause drops — Network Firewall **drops rather than passes uninspected**. For planned traffic events (launches, migrations), contact AWS Support ahead of time.

---

## Troubleshooting Matrix

| Symptom                                          | Likely cause                                             | Fix                                                                 |
| ------------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------- |
| Firewall "not scanning" / `ReceivedPackets` = 0  | Route tables don't point to the firewall endpoint         | Add/fix routes to the endpoint (VPC subnet, IGW edge, TGW route table) |
| Return traffic dropped (stateful)                | Asymmetric routing                                        | Enable TGW appliance mode; route both directions via firewall; run the symmetry test |
| High `StreamExceptionPolicyPackets`              | Asymmetric / out-of-state packets                         | Fix routing; write **paired** stateless forward+return rules         |
| Firewall endpoint won't create                   | Constrained AZ                                            | Use a supported AZ; keep resources in the same zone                  |
| Long-lived connection drops after ~350s          | GWLB idle timeout                                         | TCP keep-alive < 350s, or raise NFW idle timeout above app timeout   |
| Intermittent drops / high latency                | Burst + fail-close, NAT port exhaustion, long TCP timeouts | Check `PacketsDropped` (GWLBe), NAT `ErrorPortAllocation`, keep-alives; Support if GWLBe drops rise |
| Domain block "not working"                       | Encrypted SNI / IP access / no `Drop established` default | Add IP rules; enable TLS inspection; set drop-established default for domain lists |
| Inbound bypasses firewall                        | Missing IGW edge route table association                  | Associate a gateway route table on the IGW                          |
| Only request-side flow logs (no response)        | Return path bypasses firewall, or NACL/SG blocks inbound at server | Fix return routing; check server NACL/SG                    |

---

## Best Practices

1. **Route to the closest firewall endpoint in both directions**; never rely on cross-AZ return paths.
2. **Enable TGW appliance mode** for centralized inspection VPCs.
3. **Use the documented alert/flow test** to prove symmetry before blaming rules.
4. **Write paired stateless rules** (forward + return) so both directions forward to the stateful engine.
5. **Alarm on `ReceivedPackets`=0, `DroppedPackets`, and `StreamExceptionPolicyPackets`**.
6. **Set TCP keep-alive < 350s** (or raise NFW idle timeout) for long-lived flows.
7. **Choose the stream exception policy deliberately** (DROP for strict security, CONTINUE only when you understand the tradeoff).
8. **Place endpoints only in supported AZs**, one per AZ you serve.
9. **Pre-warm with AWS Support** before planned traffic surges (fail-close behavior).

---

## Useful Links

- [Troubleshooting general issues](https://docs.aws.amazon.com/network-firewall/latest/developerguide/troubleshooting-general-issues.html)
- [Avoiding asymmetric routing](https://docs.aws.amazon.com/network-firewall/latest/developerguide/asymmetric-routing.html)
- [Troubleshoot Network Firewall rules (re:Post)](https://repost.aws/knowledge-center/network-firewall-troubleshoot-rule-issue)
- [Fix asymmetric routing in Transit Gateway (re:Post)](https://repost.aws/knowledge-center/transit-gateway-asymmetric-route-fix)
- [TGW appliance mode](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-appliance-scenario.html)
- [Firewall policy settings (idle timeout, stream exception policy)](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-policy-settings.html)
- [Network Firewall CloudWatch metrics](https://docs.aws.amazon.com/network-firewall/latest/developerguide/monitoring-cloudwatch.html)
- [Flow operations (firewall state table)](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-flow-operations.html)

---
