# AWS Transit Gateway - Multicast Cheat Sheet

## Overview

**Transit Gateway multicast** lets a TGW act as a **multicast router**, delivering a single data stream to many receivers across **subnets of attached VPCs**. You segment multicast into **multicast domains**, associate subnets, and manage group membership either dynamically via **IGMPv2** or statically via the API/CLI.

**Key point:** Multicast is **VPC-attachment only and intra-TGW** — it does **not** traverse VPN, Direct Connect, TGW peering, or Connect attachments. You must **create a new TGW with multicast enabled** (you can't enable it on an existing TGW).

---

## Core Concepts

| Concept                 | Description                                                                 |
| ----------------------- | --------------------------------------------------------------------------- |
| **Multicast domain**    | Segments the multicast network; TGW acts as multiple multicast routers. Membership is defined at the **subnet** level |
| **Multicast group**     | A set of hosts sending/receiving the same traffic, identified by a **group IP**; membership defined by **ENIs** |
| **IGMP (IGMPv2)**        | Protocol for hosts to dynamically **JOIN/LEAVE** groups                     |
| **Multicast source**    | An ENI statically configured to **send** (static-source configs only)       |
| **Group member**        | An ENI that **receives** (in IGMP config, members can also send)            |

---

## Domain Attributes (mutually exclusive — you can't enable both)

| Attribute                | Behavior                                                                    |
| ------------------------ | --------------------------------------------------------------------------- |
| **IGMPv2 support**       | Members join/leave via IGMP JOIN/LEAVE. Non-IGMP members must be added via console/CLI (and you must deregister them; TGW ignores an IGMP LEAVE from a manually added member) |
| **Static sources support** | You register sources via `register-transit-gateway-multicast-group-sources`; **only** those sources can send |
| *(Neither enabled)*      | No designated sources — **any** instance in an associated subnet can send; group members receive |

### Membership models compared

| Model                    | Who can send                          | Who receives          | Membership managed by       | IPv6 |
| ------------------------ | ------------------------------------- | --------------------- | --------------------------- | ---- |
| **IGMP**                 | Members (send + receive)              | Members               | Hosts via IGMPv2 JOIN/LEAVE | No   |
| **Static source**        | Only registered **sources**           | Registered members    | CLI/API                     | Yes  |
| **Static group member**  | Any instance in associated subnets    | Registered members    | CLI/API                     | Yes  |

> **Only static multicast supports IPv6; dynamic (IGMP) multicast does not.**

---

## How It Works

```
Enable multicast on a NEW transit gateway
        │
        ▼
Create a multicast domain (choose IGMPv2 OR static-sources)
        │
        ▼
Associate subnets with the domain  (a subnet can be in only ONE domain)
        │
        ├── IGMP:   hosts send IGMP JOIN → TGW tracks membership
        └── Static: register sources + members via CLI
        │
        ▼
TGW replicates the stream to all group members
```

- Adding a subnet to a domain sends **all** that subnet's multicast traffic to the TGW.
- TGW issues an **IGMPv2 QUERY every 2 minutes**; members renew by replying with JOIN. Miss **3** consecutive queries → membership removed (but TGW keeps querying for **12 hours**). An explicit **LEAVE** removes immediately.
- On a TGW outage, it keeps sending data to a host for **~7 minutes (420s)** after the last JOIN.

---

## IGMP Query Traffic (SG/NACL must allow it)

TGW membership queries: **source `0.0.0.0/32`**, **destination `224.0.0.1/32`**, **protocol 2 (IGMP)**. Your security groups and NACLs must permit IGMP.

**Minimum SG/NACL rules for IGMP hosts:**

| Direction | Type / Protocol | Src/Dst                         | Purpose                 |
| --------- | --------------- | ------------------------------- | ----------------------- |
| Inbound   | IGMP (2)        | `0.0.0.0/32`                    | IGMP query              |
| Inbound   | UDP             | Remote (sender) host IP         | Inbound multicast data  |
| Outbound  | IGMP (2)        | `224.0.0.2/32`                  | IGMP leave              |
| Outbound  | IGMP (2)        | Multicast group IP              | IGMP join               |
| Outbound  | UDP             | Multicast group IP              | Outbound multicast data |

> You **can't** use SG referencing for the UDP inbound rule (must specify the sender IP); and when source+dest are in the same VPC, SG referencing to accept from the source's SG isn't supported for multicast.

---

## CLI

```bash
# Create a multicast domain (IGMPv2)
aws ec2 create-transit-gateway-multicast-domain \
  --transit-gateway-id tgw-0abc \
  --options Igmpv2Support=enable,StaticSourcesSupport=disable

# Associate a subnet + its VPC attachment with the domain
aws ec2 associate-transit-gateway-multicast-domain \
  --transit-gateway-multicast-domain-id tgw-mcast-domain-0abc \
  --transit-gateway-attachment-id tgw-attach-0vpc \
  --subnet-ids subnet-0abc

# Static source config: register a source ENI + members
aws ec2 register-transit-gateway-multicast-group-sources \
  --transit-gateway-multicast-domain-id tgw-mcast-domain-0abc \
  --group-ip-address 224.0.1.10 --network-interface-ids eni-0source

aws ec2 register-transit-gateway-multicast-group-members \
  --transit-gateway-multicast-domain-id tgw-mcast-domain-0abc \
  --group-ip-address 224.0.1.10 --network-interface-ids eni-0member1 eni-0member2

# View groups
aws ec2 search-transit-gateway-multicast-groups \
  --transit-gateway-multicast-domain-id tgw-mcast-domain-0abc
```

---

## Gotchas & Caveats

1. **Must create a NEW multicast-enabled TGW** — you can't turn multicast on for an existing TGW.
2. **VPC attachments only, intra-TGW only** — multicast does **not** route over DX, Site-to-Site VPN, TGW **peering**, or **Connect** attachments.
3. **A subnet can be in only ONE multicast domain.**
4. **Non-Nitro instances**: must disable **Source/Dest check**, and **can't be a multicast sender**.
5. **No fragmentation** — fragmented multicast packets are **dropped**; mind MTU.
6. **SG/NACL must allow IGMP** (proto 2, `0.0.0.0/32` → `224.0.0.1/32`) or membership tracking fails.
7. **IGMP JOIN loss** — if all JOIN retries are lost, the host isn't in the group; re-trigger from the app.
8. **Manually-added members ignore IGMP LEAVE** — deregister them via console/CLI (you added them, you remove them).
9. **Only static multicast supports IPv6**; IGMP (dynamic) is IPv4-only.
10. **Not for latency-sensitive workloads** (e.g., HFT) — review multicast quotas and performance with your SA.
11. **Region support is limited** — check the TGW FAQ for supported Regions.
12. **Static groups/sources auto-clean for deleted ENIs** — TGW uses its service-linked role to describe ENIs and remove stale entries.

---

## Troubleshooting

| Issue                                      | Cause                                            | Fix                                                        |
| ------------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------- |
| Host never joins the group                 | IGMP JOINs lost / SG/NACL blocks IGMP            | Allow IGMP proto 2; re-trigger JOIN from the app           |
| Members not receiving                      | Subnet not associated / wrong domain             | Associate the subnet; a subnet is in only one domain       |
| Multicast doesn't cross Regions/sites      | Not supported over peering/VPN/DX/Connect        | Multicast is intra-TGW/VPC-attachment only                 |
| Sender can't send                          | Non-Nitro sender / static-sources not registered | Use a Nitro instance; register the source ENI              |
| Packets dropped                            | Fragmentation (MTU)                              | Keep multicast packets within MTU (no fragmentation)       |
| Can't enable multicast on my TGW           | Existing TGW created without multicast           | Create a new multicast-enabled TGW                         |
| IPv6 multicast not working with IGMP       | IPv6 only supported for static multicast         | Use static source/member config for IPv6                   |

---

## Best Practices

1. **Plan multicast at TGW creation** — you can't retrofit it.
2. **Pick the membership model deliberately** (IGMP vs static-source vs static-member) based on who sends/receives and IPv6 needs.
3. **Open IGMP + UDP in SGs and NACLs** on all participating hosts.
4. **Use Nitro instances** for senders; disable Source/Dest check on any non-Nitro member.
5. **Keep packets within MTU** to avoid fragmentation drops.
6. **Review multicast quotas** and validate performance for demanding workloads.
7. **One subnet = one domain**; design domains for clean segmentation.

---

## Useful Links

- [Multicast in Transit Gateway (overview)](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html)
- [Multicast domains](https://docs.aws.amazon.com/vpc/latest/tgw/multicast-domains-about.html)
- [Managing multicast domains](https://docs.aws.amazon.com/vpc/latest/tgw/manage-domain.html)
- [Static source configurations](https://docs.aws.amazon.com/vpc/latest/tgw/multicast-configurations-no-igmp.html)
- [Static group member configurations](https://docs.aws.amazon.com/vpc/latest/tgw/multicast-configurations-no-igmp-source.html)
- [Transit Gateway quotas](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html)

---

*Sources: AWS official documentation. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
