# Amazon VPC - Endpoints & PrivateLink Cheat Sheet

## Overview

**VPC endpoints** let resources in your VPC reach AWS services (and third-party/self-hosted services via **PrivateLink**) privately, without an IGW, NAT, or public IPs. Two families exist: **Gateway endpoints** (S3, DynamoDB) and **Interface endpoints** (most other services, powered by PrivateLink).

**Key point:** Gateway endpoints are free and route via the route table; interface endpoints cost per hour + per GB and work via a private ENI/DNS.

---

## Endpoint Types

| Type                     | Backed by        | Services                          | How traffic routes                       | Cost           |
| ------------------------ | ---------------- | --------------------------------- | ---------------------------------------- | -------------- |
| **Gateway endpoint**     | Route table + prefix list | **S3 and DynamoDB only**   | Add prefix-list route to the endpoint    | **Free**       |
| **Interface endpoint**   | PrivateLink (ENI) | Most AWS services, PrivateLink services | Private ENI + private DNS in your subnets | Hourly + per-GB |
| **Gateway Load Balancer endpoint (GWLBe)** | PrivateLink | Inline appliances (firewalls) | Route table target for inspection | Hourly + per-GB |

---

## Gateway Endpoints (S3 / DynamoDB)

- Add a route: the service's **managed prefix list** (`pl-...`) → `vpce-...` in your subnet route tables.
- Controlled by an **endpoint policy** (resource policy limiting which buckets/tables/principals).
- **Regional** — only reaches S3/DynamoDB in the **same Region**.
- Free — a great way to keep S3/DynamoDB traffic off NAT.

```bash
aws ec2 create-vpc-endpoint --vpc-id vpc-0abc \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-private-1a rtb-private-1b
```

---

## Interface Endpoints (PrivateLink)

- Creates an **ENI with a private IP** in each subnet you choose, per AZ.
- **Private DNS** (optional, on by default for AWS services) overrides the public service hostname to resolve to the private ENI — so existing SDK/CLI calls transparently use the endpoint.
- Guarded by a **security group** on the endpoint ENIs and an **endpoint policy**.
- Reaches AWS services, **PrivateLink partner services**, and **your own services** exposed via an endpoint service (NLB/GWLB behind it).

```bash
aws ec2 create-vpc-endpoint --vpc-id vpc-0abc \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-1a subnet-1b \
  --security-group-ids sg-vpce \
  --private-dns-enabled
```

---

## PrivateLink: Providing Your Own Service

```
Consumer VPC                          Provider VPC
 interface endpoint (ENI) ──PrivateLink──►  VPC Endpoint Service ──► NLB/GWLB ──► your app
```

| Concept                | Detail                                                          |
| ---------------------- | --------------------------------------------------------------- |
| **Endpoint service**   | You publish a service fronted by an NLB (or GWLB)               |
| **Allowed principals** | Accounts/roles permitted to create endpoints to your service    |
| **Acceptance**         | Optionally require manual approval of each connection           |
| **Cross-account/Region** | Consumers can be in other accounts; endpoints are per-Region  |

---

## Endpoint Policies

- **Resource policy** attached to the endpoint that limits what can be accessed **through** it (e.g., only specific S3 buckets).
- Combines with IAM and the service's own resource policy — the effective permission is the **intersection**.

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-approved-bucket/*"
  }]
}
```

---

## Pricing

| Item                        | Cost                                              |
| --------------------------- | ------------------------------------------------- |
| Gateway endpoint (S3/DDB)   | **Free**                                           |
| Interface endpoint          | Per-AZ-ENU **hourly** charge + **per-GB** processed |
| GWLB endpoint               | Hourly + per-GB                                    |
| PrivateLink (provider side) | NLB/GWLB charges apply                             |

> Interface endpoints bill **per ENI per AZ per hour** — enabling an endpoint across many AZs/services adds up. Consolidate to the AZs you actually use.

---

## Quotas (defaults, adjustable)

| Resource                                | Default limit |
| --------------------------------------- | ------------- |
| Interface + gateway endpoints per VPC   | 50 (varies)   |
| Endpoints per VPC per service           | limited       |
| Endpoint services per account           | (adjustable)  |
| Connections per endpoint service        | (adjustable)  |

---

## Gotchas & Caveats

1. **Gateway endpoints only support S3 and DynamoDB** — everything else needs an interface endpoint.
2. **Gateway endpoints are same-Region only** — you can't reach S3 in another Region through them; cross-Region needs different routing.
3. **Private DNS can hijack the whole service hostname** — enabling private DNS on an interface endpoint makes *all* calls to that service resolve to the endpoint; if the endpoint SG/policy is wrong, everything to that service breaks.
4. **Endpoint security group must allow 443** from your resources — a missing SG rule causes silent timeouts.
5. **Interface endpoints are per-AZ ENIs** — if you only put the endpoint in AZ-a, resources in AZ-b traverse cross-AZ (and fail if AZ-a is down).
6. **Endpoint policy default is allow-all** — tighten it, or the endpoint permits any action the caller's IAM allows.
7. **Not all services support endpoint policies or private DNS** — check per-service support.
8. **PrivateLink endpoint services need an NLB (or GWLB)** — you can't expose an ALB directly (though NLB→ALB chaining is possible).
9. **Endpoint-service AZ mismatch** — the consumer endpoint's AZs must overlap the service's AZs, or connection fails (`does not support the Availability Zone`). AZ IDs, not names, matter across accounts.
10. **Deleting/replicating endpoints** doesn't copy policies/DNS settings — recreate them explicitly.
11. **Gateway endpoint prefix-list routes count against route-table limits** and only work for subnets whose route table has the route.
12. **On-prem access to interface endpoints** requires DNS forwarding (Resolver) so on-prem resolves the private endpoint DNS name.

---

## Troubleshooting

| Issue                                          | Cause                                           | Fix                                                        |
| ---------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| SDK/CLI to a service times out after adding endpoint | Endpoint SG blocks 443 or wrong subnet/DNS  | Allow 443 from resources; verify private DNS + subnets     |
| S3 access fails through gateway endpoint       | Missing prefix-list route or endpoint policy    | Add route to the S3 prefix list; loosen/fix policy         |
| Cross-account endpoint won't connect           | AZ mismatch or principal not allowed            | Match AZ IDs; add consumer to allowed principals           |
| Endpoint reachable only from one AZ            | ENI only created in that AZ                      | Add the endpoint to all needed subnets/AZs                 |
| On-prem can't resolve endpoint DNS             | No Resolver inbound/forwarding                   | Configure Route 53 Resolver inbound endpoint + forwarding  |
| Everything to a service breaks after enabling private DNS | Endpoint misconfig with DNS override    | Fix SG/policy/subnets or disable private DNS temporarily   |
| Endpoint service rejects connection            | Manual acceptance pending / principal missing    | Accept the connection; add the principal                   |

---

## Best Practices

1. **Use gateway endpoints for S3/DynamoDB** (free, keeps traffic off NAT).
2. **Use interface endpoints** for other AWS APIs in isolated/private subnets.
3. **Deploy interface endpoints in every AZ** you run workloads in.
4. **Lock down endpoint policies** to specific buckets/tables/actions.
5. **Restrict the endpoint SG** to 443 from known sources.
6. **Enable private DNS** for AWS services so app code needs no changes — but validate SG/policy first.
7. **For PrivateLink providers**, front with an NLB, scope allowed principals, and match AZs by AZ ID.
8. **Forward DNS from on-prem** via Resolver to consume endpoints hybrid.

---

## Useful Links

- [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Gateway endpoints (S3/DynamoDB)](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)
- [Interface endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html)
- [What is AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)
- [Endpoint services](https://docs.aws.amazon.com/vpc/latest/privatelink/create-endpoint-service.html)
- [Endpoint policies](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-access.html)
- [PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/)

---
