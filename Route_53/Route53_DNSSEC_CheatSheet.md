# AWS Route 53 DNSSEC - Cheat Sheet

## Overview

DNSSEC (Domain Name System Security Extensions) adds cryptographic signatures to DNS records, allowing resolvers to verify that responses have not been tampered with. Route 53 supports both **DNSSEC signing** (for hosted zones you own) and **DNSSEC validation** (for resolvers in your VPC).

**Purpose:** Protect against DNS spoofing, cache poisoning, and man-in-the-middle attacks.

---

## Key Concepts

+-----------------------------------+-------------------------------------------------------------------------------+
|               Term                |                                  Description                                  |
+-----------------------------------+-------------------------------------------------------------------------------+
|     **KSK** (Key-Signing Key)     |   Signs the DNSKEY record set; backed by a customer-managed key in AWS KMS    |
|    **ZSK** (Zone-Signing Key)     |  Signs all other record sets in the zone; managed automatically by Route 53   |
| **DS Record** (Delegation Signer) |         Published in the parent zone to establish the chain of trust          |
|             **RRSIG**             |            Cryptographic signature attached to each DNS record set            |
|            **DNSKEY**             |     Public keys published in DNS (contains both KSK and ZSK public keys)      |
|          **NSEC/NSEC3**           |      Authenticated denial of existence (proves a record does NOT exist)       |
|        **Chain of Trust**         | Hierarchy from root → TLD → your zone, validated via DS records at each level |
+-----------------------------------+-------------------------------------------------------------------------------+

---

## Two Sides of DNSSEC in Route 53

### 1. DNSSEC Signing (Authoritative Side)
- You enable signing on your **public hosted zone**
- Route 53 signs all responses with cryptographic signatures
- You manage the KSK; Route 53 manages the ZSK

### 2. DNSSEC Validation (Resolver Side)
- Enable on Route 53 Resolver for your VPC
- Resolver validates DNSSEC signatures on responses from signed zones
- Returns SERVFAIL for responses that fail validation

---

## Prerequisites & Requirements

+---------------------------+------------------------------------------------------------------+
|        Requirement        |                             Details                              |
+---------------------------+------------------------------------------------------------------+
|    **KMS Key Region**     |              Must be in **us-east-1** (N. Virginia)              |
|     **KMS Key Type**      |  Asymmetric, ECC_NIST_P256 key spec (signing and verification)   |
|    **KMS Permissions**    | Route 53 needs `kms:DescribeKey`, `kms:GetPublicKey`, `kms:Sign` |
|   **Hosted Zone Type**    |              Public hosted zones only (not private)              |
|   **Max KSKs per Zone**   |                                2                                 |
| **Zone must be healthy**  |            Monitor zone availability before enabling             |
| **Low TTL on NS records** |           Reduce TTL before enabling to minimize risk            |
+---------------------------+------------------------------------------------------------------+

---

## Enabling DNSSEC Signing (Step-by-Step)

### Step 1: Prepare
1. Set up CloudWatch alarms for `DNSSECInternalFailure` and `DNSSECKeySigningKeysNeedingAction`
2. Lower the SOA minimum field and NS record TTL (e.g., to 60-300 seconds)
3. Monitor zone availability
4. Wait for old TTLs to expire (let cached records with old TTL drain)

### Step 2: Enable Signing and Create KSK
1. Create (or identify) a KMS key in us-east-1 (ECC_NIST_P256, asymmetric)
2. Enable DNSSEC signing on the hosted zone
3. Route 53 creates a KSK using your KMS key and auto-generates a ZSK
4. Route 53 begins signing all records in the zone

### Step 3: Establish Chain of Trust
1. Get the DS record value from Route 53
2. Add the DS record to the **parent zone** (e.g., at your registrar or TLD)
3. Once the DS record propagates, the chain of trust is complete
4. Validating resolvers can now verify your zone's signatures

---

## Key Management

### KSK (You Manage)
- Based on your KMS customer-managed key
- Max 2 KSKs per hosted zone (for rotation purposes)
- States: `ACTION_NEEDED`, `ACTIVE`, `INACTIVE`
- You're responsible for rotation (create new KSK → update DS at parent → deactivate old KSK → delete)

### ZSK (Route 53 Manages)
- Automatically created and rotated by Route 53
- No action required from you
- Route 53 handles the signing operations

---

## DNSSEC Validation (Resolver)

Enable DNSSEC validation on Route 53 Resolver for your VPC:

- When enabled, Resolver validates signatures on responses from DNSSEC-signed zones
- If validation fails → Resolver returns **SERVFAIL** to the client
- If the zone is not signed → Resolver returns the response normally (unsigned zones are not affected)
- Only resolvers that support DNSSEC perform validation

### Enabling via Console
1. Go to Route 53 → Resolver → VPCs
2. Select your VPC
3. Check "DNSSEC validation"

---

## AWS CLI Commands

```bash
# --- DNSSEC SIGNING ---

# Create a KMS key for DNSSEC (must be in us-east-1)
aws kms create-key \
  --key-spec ECC_NIST_P256 \
  --key-usage SIGN_VERIFY \
  --description "Route53 DNSSEC KSK" \
  --region us-east-1

# Create a Key-Signing Key (KSK)
aws route53 create-key-signing-key \
  --hosted-zone-id Z1234567890ABC \
  --name "my-ksk-1" \
  --key-management-service-arn "arn:aws:kms:us-east-1:123456789012:key/abcd-1234" \
  --status ACTIVE

# Enable DNSSEC signing on a hosted zone
aws route53 enable-hosted-zone-dnssec \
  --hosted-zone-id Z1234567890ABC

# Get DNSSEC info (includes DS record for parent zone)
aws route53 get-dnssec \
  --hosted-zone-id Z1234567890ABC

# Disable DNSSEC signing
aws route53 disable-hosted-zone-dnssec \
  --hosted-zone-id Z1234567890ABC

# Deactivate a KSK (must deactivate before deleting)
aws route53 deactivate-key-signing-key \
  --hosted-zone-id Z1234567890ABC \
  --name "my-ksk-1"

# Delete a KSK (must be inactive first)
aws route53 delete-key-signing-key \
  --hosted-zone-id Z1234567890ABC \
  --name "my-ksk-1"

# --- DNSSEC VALIDATION (Resolver) ---

# Update Resolver config to enable DNSSEC validation
aws route53resolver update-resolver-dnssec-config \
  --resource-id vpc-0123456789abcdef0 \
  --validation ENABLE

# Get DNSSEC validation config for a VPC
aws route53resolver get-resolver-dnssec-config \
  --resource-id vpc-0123456789abcdef0

# List all DNSSEC validation configs
aws route53resolver list-resolver-dnssec-configs
```

---

## CloudWatch Monitoring

### Critical Alarms (Set These Up BEFORE Enabling DNSSEC)

+-------------------------------------+-------------+---------------------------------------------+
|               Metric                |  Namespace  |                 Description                 |
+-------------------------------------+-------------+---------------------------------------------+
|       `DNSSECInternalFailure`       | AWS/Route53 | Route 53 internal error with DNSSEC signing |
| `DNSSECKeySigningKeysNeedingAction` | AWS/Route53 |   KSK needs attention (rotation, expiry)    |
+-------------------------------------+-------------+---------------------------------------------+

```bash
# Create alarm for DNSSEC internal failures
aws cloudwatch put-metric-alarm \
  --alarm-name "DNSSEC-Internal-Failure" \
  --namespace AWS/Route53 \
  --metric-name DNSSECInternalFailure \
  --dimensions Name=HostedZoneId,Value=Z1234567890ABC \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-topic \
  --region us-east-1
```

---

## Disabling DNSSEC Safely

**Order matters. Incorrect order can cause zone outage.**

1. Remove the DS record from the parent zone
2. Wait for DS record TTL to expire (verify propagation)
3. Disable DNSSEC signing on the hosted zone
4. Deactivate and delete KSKs

> Route 53 checks if the zone is still in the chain of trust before allowing disable. If DS records still exist at the parent, disabling will cause validation failures.

---

## Common Issues & Troubleshooting

+-------------------------------------------+---------------------------------------------+-----------------------------------------------+
|                   Issue                   |                    Cause                    |                      Fix                      |
+-------------------------------------------+---------------------------------------------+-----------------------------------------------+
|  Zone becomes unreachable after enabling  |  DS record added before signing was active  |       Enable signing first, then add DS       |
|            SERVFAIL responses             | Signatures invalid or chain of trust broken |     Check KSK status, verify DS at parent     |
| `DNSSECKeySigningKeysNeedingAction` alarm |     KSK needs rotation or KMS key issue     |       Check KMS key status, rotate KSK        |
|           Cannot disable DNSSEC           |      DS record still present at parent      |  Remove DS first, wait for TTL, then disable  |
|         KMS key deleted/disabled          |              Signing will fail              | Restore KMS key immediately or create new KSK |
+-------------------------------------------+---------------------------------------------+-----------------------------------------------+

---

## Limitations

+------------------------------+-------------------------------------------------------------+
|          Limitation          |                           Details                           |
+------------------------------+-------------------------------------------------------------+
|   Public hosted zones only   |              Cannot sign private hosted zones               |
| KMS key must be in us-east-1 |          Even if hosted zone serves global traffic          |
|     Max 2 KSKs per zone      |              Designed for rotation (old + new)              |
|          Algorithm           |        ECC_NIST_P256 only (ECDSA P-256 with SHA-256)        |
|  Not all resolvers validate  |       Only DNSSEC-aware resolvers perform validation        |
|    Response size increase    |     DNSSEC adds RRSIG records, increasing response size     |
|        NSEC vs NSEC3         | Route 53 uses NSEC (reveals zone contents via zone walking) |
+------------------------------+-------------------------------------------------------------+

---

## Best Practices

1. **Always set up CloudWatch alarms before enabling** - `DNSSECInternalFailure` and `DNSSECKeySigningKeysNeedingAction`
2. **Lower TTLs before enabling** - Reduce SOA minimum and NS TTL so you can recover quickly if issues arise
3. **Test with a non-production zone first** - Validate your process before touching production domains
4. **Monitor after adding DS record** - The chain of trust activation is the riskiest step
5. **Don't delete KMS keys** - Use key rotation instead; deleting the KMS key breaks signing permanently
6. **Plan KSK rotation** - Create a second KSK, update DS at parent, then remove old KSK
7. **Use Route 53 Profiles** - For consistent DNSSEC validation across multiple VPCs/accounts

---

## Pricing

+-------------------+--------------------------------------------------------------+
|       Item        |                             Cost                             |
+-------------------+--------------------------------------------------------------+
|  DNSSEC signing   |               No additional charge for signing               |
|      KMS key      | Standard KMS pricing applies (~$1/month per key + API calls) |
| DNSSEC validation |                     No additional charge                     |
+-------------------+--------------------------------------------------------------+

> The main cost is the KMS asymmetric key in us-east-1.

---

## Useful Links

- [Configuring DNSSEC Signing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec.html)
- [Enabling DNSSEC Signing & Chain of Trust](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec-enable-signing.html)
- [Working with KSKs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec-ksk.html)
- [KMS Key & ZSK Management](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec-zsk-management.html)
- [KMS Key Requirements for DNSSEC](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec-cmk-requirements.html)
- [DNSSEC Validation in Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dnssec-validation.html)
- [Troubleshooting DNSSEC](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec-troubleshoot.html)
- [AWS Blog: DNSSEC Signing and Validation](https://aws.amazon.com/blogs/networking-and-content-delivery/configuring-dnssec-signing-and-validation-with-amazon-route-53/)

---