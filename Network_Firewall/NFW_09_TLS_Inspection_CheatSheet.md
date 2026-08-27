# AWS Network Firewall - TLS Inspection Cheat Sheet

## Overview

**TLS inspection** lets Network Firewall decrypt, inspect, and re-encrypt TLS traffic so stateful rules can see inside otherwise-encrypted flows (payload content, full domain, Suricata content matches). It's configured via a **TLS inspection configuration** referenced by the firewall policy, using an **ACM certificate** for the man-in-the-middle.

**Key point:** Without TLS inspection, domain filtering sees only the **cleartext SNI/Host** — encrypted-SNI or content-based rules can't inspect the payload. TLS inspection adds significant capability but also cost, latency, and certificate-management complexity.

---

## Core Concepts

| Element                       | Description                                                    |
| ----------------------------- | -------------------------------------------------------------- |
| **TLS inspection configuration** | Defines what to decrypt (scope), which cert, and behaviors  |
| **ACM certificate**           | The CA/cert used to re-sign traffic (clients must trust it for outbound MITM) |
| **Scope**                     | 5-tuple criteria selecting which flows to decrypt              |
| **Inbound vs outbound**       | Inbound (server cert in ACM) vs outbound (CA to re-sign)       |
| **Certificate revocation**    | Optional CRL/OCSP-style checks on server certs                 |

---

## Inbound vs Outbound TLS Inspection

| Direction  | Use case                              | Certificate                                    |
| ---------- | ------------------------------------- | ---------------------------------------------- |
| **Inbound**| Inspect TLS to your own servers       | Import your server certificate into ACM        |
| **Outbound**| Inspect TLS from clients to internet | A CA certificate in ACM; **clients must trust** it to avoid TLS errors |

---

## How It Works

```
Client → TLS ClientHello → Firewall (TLS inspection config scope match?)
        |
        ├── in scope → firewall terminates TLS (using ACM cert), decrypts,
        |              runs stateful rules on cleartext, re-encrypts to origin
        └── not in scope → passes through encrypted (SNI/Host only)
```

---

## CLI

```bash
# Create a TLS inspection configuration (outbound, CA cert in ACM)
aws network-firewall create-tls-inspection-configuration \
  --tls-inspection-configuration-name outbound-tls \
  --tls-inspection-configuration '{
    "ServerCertificateConfigurations": [{
      "CertificateAuthorityArn": "arn:aws:acm-pca:...:certificate-authority/abc",
      "Scopes": [{
        "Protocols": [6],
        "DestinationPorts": [{"FromPort":443,"ToPort":443}]
      }]
    }]
  }'

# Reference it from the firewall policy (TLSInspectionConfigurationArn)
aws network-firewall update-firewall-policy \
  --firewall-policy-name my-policy \
  --firewall-policy '{"...":"...","TLSInspectionConfigurationArn":"arn:...:tls-configuration/outbound-tls"}'
```

---

## Pricing

| Item                     | Cost                                          |
| ------------------------ | --------------------------------------------- |
| TLS inspection           | Additional per-GB decrypted/inspected (on top of standard NFW data processing) |
| ACM / ACM PCA            | Certificate authority costs (for outbound CA) |

> TLS inspection adds a **per-GB inspection cost** and can add latency — scope it to the flows that truly need content inspection rather than all 443 traffic.

---

## Gotchas & Caveats

1. **Outbound inspection is a MITM** — clients must **trust the CA** in ACM, or they get TLS/certificate errors. Distribute the CA to client trust stores.
2. **Not everything is inspectable** — pinned certificates, mutual TLS (mTLS), and some protocols will fail or must be excluded from scope.
3. **Scope carefully** — inspecting all 443 traffic is costly and can break apps that use cert pinning; scope to specific CIDRs/ports/destinations.
4. **Certificate lifecycle** — the ACM/PCA certificate must be valid and renewed; expiry breaks inspection (and possibly traffic).
5. **Latency & cost** — decryption/re-encryption adds processing time and per-GB fees.
6. **Revocation checks** can drop flows to servers with revoked/invalid certs — understand the behavior before enabling.
7. **Inbound vs outbound need different certs** — inbound uses your server cert; outbound uses a CA to re-sign.
8. **Some regions/features** for TLS inspection rolled out later — verify availability.
9. **Compliance/privacy implications** — decrypting user traffic has legal/privacy considerations; document and get approval.
10. **Interaction with SNI-based domain rules** — with TLS inspection you can match full content, but ensure rules are updated to use decrypted context.
11. **Troubleshooting is harder** — TLS handshake failures may originate from scope/cert issues rather than the app.
12. **Not a substitute for endpoint controls** — inspection covers in-transit; it doesn't replace host security.

---

## Troubleshooting

| Issue                                       | Cause                                          | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Clients get certificate errors              | CA not trusted by clients (outbound MITM)       | Distribute the ACM CA to client trust stores               |
| Some sites break with inspection on         | Cert pinning / mTLS                             | Exclude those destinations from the inspection scope       |
| Inspection stopped working                  | Expired ACM/PCA certificate                     | Renew/replace the certificate                              |
| High cost/latency                           | Inspecting all 443 traffic                      | Narrow the scope to needed flows                           |
| Flows to some servers dropped               | Revocation check on invalid server cert         | Review revocation settings                                 |
| Rules still don't match content             | Rules not using decrypted context               | Update stateful rules to inspect payload/content           |

---

## Best Practices

1. **Scope TLS inspection tightly** to destinations/ports that need content inspection.
2. **Distribute the outbound CA** to all client trust stores before enabling.
3. **Exclude pinned/mTLS destinations** from scope.
4. **Manage certificate lifecycle** (renewal alarms) to avoid outages.
5. **Weigh cost, latency, and privacy** — enable only where the security benefit justifies it.
6. **Test in a staging firewall** before production rollout.
7. **Update stateful rules** to leverage decrypted content once inspection is on.
8. **Document compliance approval** for decrypting traffic.

---

## Useful Links

- [TLS inspection in Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/tls-inspection.html)
- [TLS inspection configurations](https://docs.aws.amazon.com/network-firewall/latest/developerguide/tls-inspection-configurations.html)
- [Requirements & certificates](https://docs.aws.amazon.com/network-firewall/latest/developerguide/tls-inspection-certificate-requirements.html)
- [Scope configuration](https://docs.aws.amazon.com/network-firewall/latest/developerguide/tls-inspection-scope.html)
- [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)

---
