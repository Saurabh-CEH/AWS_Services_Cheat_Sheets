# AWS Route 53 Domains - Cheat Sheet

## Overview

Route 53 is an ICANN-accredited domain registrar that allows you to register, transfer, and manage domain names directly within AWS. When you register a domain through Route 53, it automatically creates a public hosted zone and configures the NS records at the registry.

**Key point:** Domain registration and hosted zones are separate concepts. You can register domains with Route 53 but use another DNS provider, or use Route 53 DNS with domains registered elsewhere.

---

## Core Concepts

| Component                    | Description                                                                |
| ---------------------------- | -------------------------------------------------------------------------- |
| **Domain Registration**      | The act of reserving a domain name with a registrar                        |
| **Registrar**                | Organization accredited to sell domains (Route 53 is a registrar)          |
| **Registry**                 | Organization that maintains the TLD database (.com, .net, etc.)            |
| **Registrant**               | The person/organization that owns the domain                               |
| **EPP Code (Auth Code)**     | Transfer authorization code required to move domains between registrars    |
| **Domain Lock**              | Prevents unauthorized transfers (Transfer Lock)                            |
| **WHOIS Privacy**            | Hides registrant contact info from public WHOIS (free with Route 53)       |
| **Name Servers**             | DNS servers that the registry delegates your domain to                     |
| **Auto-Renew**               | Automatic domain renewal before expiration                                 |

---

## Domain Registration Workflow

```
1. Check domain availability
        |
        v
2. Register domain (provide contact info)
        |
        v
3. Route 53 creates public hosted zone automatically
        |
        v
4. NS records configured at registry pointing to Route 53
        |
        v
5. Domain is active (may take up to 48 hours for new TLDs)
        |
        v
6. Email verification (required for some TLDs)
```

---

## EPP Status Codes

| Code                       | Meaning                                                  |
| -------------------------- | -------------------------------------------------------- |
| `clientTransferProhibited` | Transfer lock enabled (you set this)                     |
| `clientDeleteProhibited`   | Domain cannot be deleted (you set this)                  |
| `clientUpdateProhibited`   | Domain contact/NS cannot be modified (you set this)      |
| `serverTransferProhibited` | Registry-level transfer lock (registry sets this)        |
| `serverDeleteProhibited`   | Registry prevents deletion (registry sets this)          |
| `pendingTransfer`          | Domain transfer is in progress                           |
| `pendingDelete`            | Domain is in the deletion grace period                   |
| `redemptionPeriod`         | Domain expired and is in redemption (can restore for fee)|
| `ok`                       | Domain is active with no restrictions                    |

---

## Domain Transfer

### Transfer INTO Route 53

```
1. Unlock domain at current registrar
2. Disable WHOIS privacy at current registrar (if required)
3. Obtain EPP/Auth code from current registrar
4. Initiate transfer in Route 53 console
5. Confirm transfer via email
6. Wait 5-7 days (some TLDs are faster)
7. Route 53 creates hosted zone (or uses existing one)
```

### Transfer OUT of Route 53

```
1. Disable transfer lock in Route 53
2. Obtain EPP/Auth code from Route 53
3. Initiate transfer at new registrar
4. Approve transfer via email (or wait for auto-approval after 5 days)
```

### Transfer Rules

| Rule                                | Details                                           |
| ----------------------------------- | ------------------------------------------------- |
| Minimum age for transfer            | 60 days after registration or last transfer       |
| Lock after transfer                 | 60-day lock at new registrar (ICANN requirement)  |
| Expiration extension                | Transfer adds 1 year to expiration (most TLDs)    |
| Domains within 15 days of expiry    | Some registries block transfers                   |
| Recently renewed (within 45 days)   | Extra year may not be added                       |

---

## Domain Pricing

| TLD         | Registration (1yr) | Renewal (1yr) | Transfer   |
| ----------- | ------------------ | ------------- | ---------- |
| `.com`      | $13.00             | $13.00        | $13.00     |
| `.net`      | $11.00             | $11.00        | $11.00     |
| `.org`      | $12.00             | $12.00        | $12.00     |
| `.io`       | $39.00             | $39.00        | $39.00     |
| `.dev`      | $14.00             | $14.00        | $14.00     |
| `.cloud`    | $12.00             | $12.00        | N/A        |
| `.app`      | $14.00             | $14.00        | $14.00     |

> Prices vary by TLD and change over time. Check the [Route 53 pricing page](https://d32ze2gidvkk54.cloudfront.net/Amazon_Route_53_Domain_Registration_Pricing_20140731.csv) for current pricing.

---

## Domain Contact Types

| Contact Type    | Purpose                                            |
| --------------- | -------------------------------------------------- |
| **Registrant**  | Legal owner of the domain                          |
| **Admin**       | Administrative contact for domain matters          |
| **Tech**        | Technical contact for DNS and server issues         |

> Route 53 allows you to use the same contact info for all three, or specify different contacts.

---

## WHOIS Privacy Protection

- **Free** with Route 53 domain registration
- Hides personal contact information from public WHOIS queries
- Enabled by default for supported TLDs
- Some TLDs (country-code TLDs) may not support privacy protection
- Does not affect ICANN's ability to contact you or legal processes

---

## Domain Expiration Timeline

```
Domain Active
    |
    v (Expiration Date)
Grace Period (0-45 days depending on TLD)
    - Domain still works
    - Can renew at normal price
    |
    v
Redemption Period (30 days typical)
    - Domain stops resolving
    - Can restore for a redemption fee ($80-$200+)
    |
    v
Pending Delete (5 days)
    - Cannot be recovered
    - Will be released for public registration
    |
    v
Available for anyone to register
```

---

## AWS CLI Commands

```bash
# --- DOMAIN REGISTRATION ---

# Check domain availability
aws route53domains check-domain-availability \
  --domain-name example.com

# Register a domain
aws route53domains register-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --admin-contact FirstName=John,LastName=Doe,ContactType=PERSON,\
OrganizationName=MyCompany,AddressLine1="123 Main St",City=Seattle,\
State=WA,CountryCode=US,ZipCode=98101,PhoneNumber=+1.2065551234,\
Email=admin@example.com \
  --registrant-contact FirstName=John,LastName=Doe,ContactType=PERSON,\
OrganizationName=MyCompany,AddressLine1="123 Main St",City=Seattle,\
State=WA,CountryCode=US,ZipCode=98101,PhoneNumber=+1.2065551234,\
Email=admin@example.com \
  --tech-contact FirstName=John,LastName=Doe,ContactType=PERSON,\
OrganizationName=MyCompany,AddressLine1="123 Main St",City=Seattle,\
State=WA,CountryCode=US,ZipCode=98101,PhoneNumber=+1.2065551234,\
Email=admin@example.com \
  --auto-renew

# --- DOMAIN MANAGEMENT ---

# List registered domains
aws route53domains list-domains

# Get domain detail
aws route53domains get-domain-detail --domain-name example.com

# Enable auto-renew
aws route53domains enable-domain-auto-renew --domain-name example.com

# Disable auto-renew
aws route53domains disable-domain-auto-renew --domain-name example.com

# Renew domain
aws route53domains renew-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --current-expiry-year 2025

# --- TRANSFER LOCK ---

# Enable transfer lock
aws route53domains enable-domain-transfer-lock --domain-name example.com

# Disable transfer lock
aws route53domains disable-domain-transfer-lock --domain-name example.com

# --- WHOIS PRIVACY ---

# Get contact info (privacy status)
aws route53domains get-domain-detail --domain-name example.com

# Update privacy (hide contact info)
aws route53domains update-domain-contact-privacy \
  --domain-name example.com \
  --admin-privacy \
  --registrant-privacy \
  --tech-privacy

# --- TRANSFER ---

# Transfer domain INTO Route 53
aws route53domains transfer-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --auth-code "EPP_AUTH_CODE_HERE" \
  --admin-contact FirstName=John,LastName=Doe,ContactType=PERSON,\
Email=admin@example.com,PhoneNumber=+1.2065551234,\
AddressLine1="123 Main St",City=Seattle,State=WA,\
CountryCode=US,ZipCode=98101 \
  --registrant-contact FirstName=John,LastName=Doe,ContactType=PERSON,\
Email=admin@example.com,PhoneNumber=+1.2065551234,\
AddressLine1="123 Main St",City=Seattle,State=WA,\
CountryCode=US,ZipCode=98101 \
  --tech-contact FirstName=John,LastName=Doe,ContactType=PERSON,\
Email=admin@example.com,PhoneNumber=+1.2065551234,\
AddressLine1="123 Main St",City=Seattle,State=WA,\
CountryCode=US,ZipCode=98101 \
  --auto-renew

# Get EPP/Auth code (for transfer OUT)
aws route53domains retrieve-domain-auth-code --domain-name example.com

# Check transfer status
aws route53domains get-operation-detail --operation-id op-id-here

# --- NAME SERVERS ---

# Update name servers for a domain
aws route53domains update-domain-nameservers \
  --domain-name example.com \
  --nameservers Name=ns-1234.awsdns-56.org Name=ns-789.awsdns-01.co.uk \
Name=ns-456.awsdns-23.com Name=ns-012.awsdns-78.net

# --- TAGS ---

# Tag a domain
aws route53domains update-tags-for-domain \
  --domain-name example.com \
  --tags-to-update Key=Environment,Value=Production Key=Team,Value=Platform

# List tags
aws route53domains list-tags-for-domain --domain-name example.com

# --- OPERATIONS ---

# List recent operations (registrations, transfers, renewals)
aws route53domains list-operations

# Get operation detail
aws route53domains get-operation-detail --operation-id op-abcdef1234
```

---

## Domain Registration API Region

| Operation             | API Region                                  |
| --------------------- | ------------------------------------------- |
| Domain registration   | `us-east-1` only (global service endpoint)  |
| Domain management     | `us-east-1` only                            |
| Hosted zone management| Any region (but Route 53 is global)         |

> All `route53domains` CLI commands must target `us-east-1` regardless of where your resources are.

---

## Supported TLDs

Route 53 supports 300+ TLDs including:

| Category              | Examples                                            |
| --------------------- | --------------------------------------------------- |
| Generic (gTLDs)       | .com, .net, .org, .info, .biz                       |
| New gTLDs             | .app, .dev, .cloud, .io, .ai, .tech, .store        |
| Country Code (ccTLDs) | .co.uk, .de, .fr, .jp, .au, .ca, .in               |
| Geographic            | .berlin, .london, .nyc, .tokyo                      |
| Industry              | .bank, .insurance, .law, .medical                   |

> Not all TLDs support all features (privacy, transfer lock, DNSSEC). Check AWS documentation for TLD-specific restrictions.

---

## Quotas

| Resource                          | Default Limit                     |
| --------------------------------- | --------------------------------- |
| Domains per account               | 20 (soft limit, can increase)     |
| Domain registration period        | 1-10 years (varies by TLD)        |
| Name servers per domain           | Up to 13 (max for DNS protocol)   |
| Contact fields max length         | Varies (typically 255 chars)      |

---

## Troubleshooting

| Issue                                  | Cause                                          | Fix                                                        |
| -------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Domain not resolving after registration| DNS propagation delay                          | Wait up to 48 hours; verify NS at registrar                |
| Transfer rejected                      | Domain locked or within 60-day restriction     | Unlock domain; ensure 60 days since last transfer          |
| Transfer stuck pending                 | Email confirmation not completed               | Check registrant email for approval link                   |
| Cannot get EPP code                    | Some TLDs don't use EPP codes                  | Check TLD-specific transfer process                        |
| Domain expired unexpectedly            | Auto-renew disabled or payment failed          | Enable auto-renew; update payment method                   |
| WHOIS still showing info after privacy | Privacy not supported for that TLD             | Check TLD privacy support in AWS docs                      |
| Cannot register domain                 | Domain taken or TLD not supported              | Try variations or different TLD                            |
| Email verification failing             | Wrong email or email not received              | Check spam; update contact email; resend verification      |

---

## Best Practices

1. **Enable transfer lock** — Prevents unauthorized domain transfers (social engineering attacks)
2. **Enable auto-renew** — Prevent accidental expiration of critical domains
3. **Use valid contact email** — Required for verification, transfer approvals, and ICANN notices
4. **Enable WHOIS privacy** — Reduces spam and protects personal information
5. **Register for multiple years** — Ensures you don't lose important domains due to renewal failures
6. **Monitor expiration dates** — Set calendar reminders as backup to auto-renew
7. **Keep payment method current** — Failed payments can result in domain loss
8. **Register common variations** — Prevent typosquatting (.com, .net, .org, common misspellings)
9. **Use AWS Organizations consolidated billing** — Centralize domain costs
10. **Document domain ownership** — Maintain records of who owns what, especially in multi-team orgs
11. **Plan transfers carefully** — Factor in 60-day lock periods and propagation time
12. **Use CloudTrail** — Audit all domain registration API calls

---

## Integration with AWS Services

| Service              | Integration                                                    |
| -------------------- | -------------------------------------------------------------- |
| **Route 53 DNS**     | Auto-creates hosted zone on registration                       |
| **ACM**             | DNS validation for SSL/TLS certificates                        |
| **CloudTrail**      | Audits all domain registration operations                      |
| **AWS Budgets**     | Track domain registration costs                                |
| **Organizations**   | Consolidated billing for domains across accounts               |
| **IAM**            | Control who can register/manage domains                         |

---

## Useful Links

- [Registering Domains](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/registrar.html)
- [Transferring Domains](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-transfer.html)
- [Domain Pricing](https://d32ze2gidvkk54.cloudfront.net/Amazon_Route_53_Domain_Registration_Pricing_20140731.csv)
- [Supported TLDs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/registrar-tld-list.html)
- [Domain Lock/Unlock](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-lock.html)
- [Renewing Domains](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-renew.html)
- [WHOIS Privacy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-privacy-protection.html)
- [EPP Status Codes (ICANN)](https://www.icann.org/resources/pages/epp-status-codes-2014-06-16-en)

---
