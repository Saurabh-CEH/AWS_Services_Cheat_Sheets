# AWS Transit Gateway - Sharing & Multi-Account Cheat Sheet

## Overview

In multi-account environments a Transit Gateway is typically owned by a **central networking account** and **shared** to workload accounts via **AWS Resource Access Manager (RAM)**. Shared-with accounts then create their own attachments to the TGW without owning it.

**Key point:** Sharing a TGW via RAM lets other accounts **attach** to it — but the **route tables stay owned/managed by the TGW owner**. Consumers can't edit TGW routing; that's centralized by design.

---

## Sharing Model

| Role                    | Can do                                                              |
| ----------------------- | ------------------------------------------------------------------- |
| **TGW owner (networking acct)** | Create/manage TGW, route tables, associations, propagations, peering |
| **Shared-with account** | Create VPC attachments to the shared TGW (accept if required)       |
| **RAM resource share**  | The mechanism that grants access to the TGW                         |
| **Auto-accept**         | TGW option to auto-accept attachment requests from shared accounts  |

---

## How It Works

```
Networking account: create TGW → create RAM share → add principals (accounts/OUs)
        |
        v
Workload account: sees the shared TGW → creates a VPC attachment
        |
        ├── TGW auto-accept ON  → attachment active immediately
        └── auto-accept OFF     → owner must accept the attachment
        |
        v
Owner associates/propagates the attachment in the appropriate TGW route table
```

> Attachment **creation** is done by the consumer; **routing** (which segment it lands in) is controlled by the owner via association/propagation.

---

## CLI

```bash
# (Owner) Share the TGW via RAM to an OU or accounts
aws ram create-resource-share \
  --name "tgw-share" \
  --resource-arns arn:aws:ec2:us-east-1:111122223333:transit-gateway/tgw-0abc \
  --principals ou-abcd-12345678

# (Owner) Optionally require manual acceptance of attachments
aws ec2 modify-transit-gateway \
  --transit-gateway-id tgw-0abc \
  --options AutoAcceptSharedAttachments=disable

# (Consumer) Create a VPC attachment to the shared TGW
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-0abc --vpc-id vpc-consumer \
  --subnet-ids subnet-a subnet-b

# (Owner) Accept the attachment if auto-accept is off
aws ec2 accept-transit-gateway-vpc-attachment \
  --transit-gateway-attachment-id tgw-attach-consumer

# (Owner) Associate + propagate into the right route table (segmentation)
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id tgw-rtb-prod \
  --transit-gateway-attachment-id tgw-attach-consumer
```

---

## Gotchas & Caveats

1. **RAM sharing is required for cross-account attachments** — a workload account can't attach to a TGW it can't see.
2. **RAM sharing to OUs requires Organizations trusted access** for RAM enabled — otherwise share by explicit account IDs.
3. **Consumers can create attachments but not manage routing** — association/propagation is the owner's job; consumers can't self-service into a segment.
4. **Auto-accept vs manual accept** — with auto-accept off, attachments sit pending until the owner accepts; with it on, any shared account can attach without review (governance tradeoff).
5. **Owner must associate/propagate new attachments** — a freshly accepted attachment has no connectivity until the owner puts it in a route table.
6. **Deleting the RAM share doesn't delete existing attachments** — they persist; clean up explicitly.
7. **Security group referencing across accounts** over TGW is limited — plan for CIDR-based rules.
8. **Billing** — attachment-hour and data-processing charges accrue to the attachment owner/account per AWS billing rules; clarify cost ownership.
9. **Consumer subnet route tables still need `→ tgw`** — sharing doesn't add VPC-side routes.
10. **Cross-account peering also needs acceptance** — sharing (RAM) is for attachments; peering has its own accept flow.
11. **Quota interactions** — many shared accounts attaching can approach the per-TGW attachment limit (5,000).
12. **Tag/visibility** — shared resources appear in consumer accounts but with limited management; set clear naming so teams know it's centrally owned.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Consumer can't see the TGW                  | Not shared via RAM / wrong principals           | Create/adjust the RAM share; add the account/OU            |
| Attachment stuck pending                    | Auto-accept off, owner hasn't accepted          | Owner accepts the attachment                               |
| Attachment active but no connectivity       | Owner hasn't associated/propagated it           | Owner adds it to the right route table                     |
| Consumer can't edit routes                  | By design — owner manages routing               | Request routing change from networking team                |
| Share to OU fails                           | RAM Organizations trusted access disabled       | Enable RAM sharing with Organizations                      |
| Orphaned attachments after share removal    | Removing share doesn't delete attachments       | Delete attachments explicitly                              |

---

## Best Practices

1. **Own the TGW in a central networking account** and share via RAM to OUs.
2. **Keep auto-accept OFF** for governance, and formalize an attachment-request/approval process.
3. **Owner controls segmentation** — associate/propagate new attachments into the correct route table.
4. **Enable RAM Organizations sharing** to target OUs cleanly.
5. **Document cost ownership** for attachment-hours and data processing.
6. **Use consistent naming/tags** so workload teams recognize centrally shared TGWs.
7. **Clean up attachments** before removing shares; audit periodically.

---

## Useful Links

- [Sharing a transit gateway (RAM)](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-transit-gateways.html#tgw-sharing)
- [AWS RAM](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html)
- [Multi-account TGW (whitepaper)](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-gateway.html)
- [Accept shared attachments](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)

---
