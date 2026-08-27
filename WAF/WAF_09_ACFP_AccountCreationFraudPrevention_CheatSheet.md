# AWS WAF - Account Creation Fraud Prevention (ACFP) Cheat Sheet

## Overview

**Account Creation Fraud Prevention (ACFP)** is a paid managed rule group (`AWSManagedRulesACFPRuleSet`) that protects **sign-up / registration** endpoints from mass fake-account creation, bulk bot registrations, and use of stolen/compromised data during sign-up.

**Key point:** ACFP is the sign-up counterpart to ATP. It inspects the **registration page**, the **account-creation request**, and optionally the **response**. Like ATP, it fails silently if the paths or field pointers are wrong.

---

## What ACFP Detects

| Capability                          | Description                                                             |
| ----------------------------------- | ----------------------------------------------------------------------- |
| **Automated / bulk sign-ups**       | Bots creating many accounts                                             |
| **Compromised credential use**      | Email/password from breach data submitted at sign-up                    |
| **Suspicious account attributes**   | Anomalous email/phone/address patterns                                  |
| **Volumetric registration**         | Too many sign-ups per IP / session                                      |
| **Registration response analysis**  | Reads sign-up success/failure from origin (optional)                    |
| **Navigation check**                | Verifies the user actually visited the registration page first          |

---

## Required Configuration

ACFP needs both the **registration page path** (where the form loads) and the **creation path** (where the account is submitted), plus the payload shape:

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesACFPRuleSet",
    "ManagedRuleGroupConfigs": [
      {
        "AWSManagedRulesACFPRuleSet": {
          "CreationPath": "/api/signup",
          "RegistrationPagePath": "/signup",
          "RequestInspection": {
            "PayloadType": "JSON",
            "UsernameField": { "Identifier": "/email" },
            "PasswordField": { "Identifier": "/password" },
            "EmailField": { "Identifier": "/email" },
            "PhoneNumberFields": [ { "Identifier": "/phone" } ],
            "AddressFields": [ { "Identifier": "/addr1" }, { "Identifier": "/city" } ]
          },
          "ResponseInspection": {
            "StatusCode": { "SuccessCodes": [200], "FailureCodes": [400, 409] }
          },
          "EnableRegexInPath": false
        }
      }
    ]
  }
}
```

| Field                     | Notes                                                                       |
| ------------------------- | --------------------------------------------------------------------------- |
| **CreationPath**          | URI that submits/creates the account (the POST endpoint)                    |
| **RegistrationPagePath**  | URI that serves the sign-up form (used for navigation/interstitial checks)  |
| **PayloadType**           | `JSON` or `FORM_ENCODED` — must match the client                            |
| **UsernameField / PasswordField / EmailField** | Pointers/field names for those attributes              |
| **PhoneNumberFields / AddressFields** | Optional; enable richer fraud attribute checks                  |
| **ResponseInspection**    | Optional; CloudFront & ALB only                                             |

> **Gotcha:** ACFP requires **both** `CreationPath` and `RegistrationPagePath`. The registration page path drives navigation/interstitial checks — omit or mis-set it and those protections don't function.

---

## Response Inspection (Optional)

Same mechanism and constraints as ATP — reads the origin's sign-up result to know if an account was actually created.

| Inspection type | Identifies success/failure via     |
| --------------- | ---------------------------------- |
| **StatusCode**  | Success vs failure HTTP codes      |
| **Header**      | Header name + values               |
| **BodyContains**| Strings in the response body       |
| **Json**        | JSON field + values                |

> **Gotcha:** Response inspection is **CloudFront/ALB only**. On API Gateway, AppSync, Cognito, App Runner it has no effect.

---

## ACFP Labels

| Label (representative)                                              | Meaning                             |
| ------------------------------------------------------------------ | ----------------------------------- |
| `awswaf:managed:aws:acfp:signal:credential_compromised`            | Compromised credential at sign-up   |
| `awswaf:managed:aws:acfp:signal:volumetric_*`                      | Volumetric registration signal      |
| `awswaf:managed:aws:acfp:aggregate:*`                              | Aggregated abuse signals            |
| `awswaf:managed:aws:acfp:signal:automated_browser`                 | Automated browser at sign-up        |
| `awswaf:managed:token:absent` / `rejected`                          | Token state                         |

```
Priority 30:  ACFP rule group        (adds labels)
Priority 31:  Block → LabelMatch acfp:signal:credential_compromised
Priority 32:  Challenge → LabelMatch acfp:signal:automated_browser
```

---

## Rules Inside the ACFP Group (examples)

| Rule name                          | Action tendency | Purpose                                      |
| ---------------------------------- | --------------- | -------------------------------------------- |
| `VolumetricIpHigh`                 | Block/Challenge | Excessive sign-ups per IP                    |
| `VolumetricSession`                | Challenge       | Excessive sign-ups per session               |
| `AttributeCompromisedCredentials`  | Block           | Breached email/password at sign-up           |
| `AttributeUsernameTraversal`       | Block           | Username field manipulation                  |
| `RiskScore*` / ML signals          | Varies          | Coordinated fraud detection                  |
| `MissingCredential`                | Block/Count     | Malformed sign-up request                    |

> **Gotcha:** ACFP includes **token-dependent** rules. Without the JS/Mobile SDK, legitimate sign-ups may be treated as token-absent and challenged/blocked.

---

## ATP vs ACFP — Quick Contrast

| Aspect            | ATP                          | ACFP                                   |
| ----------------- | ---------------------------- | -------------------------------------- |
| Protects          | Login / sign-in              | Registration / sign-up                 |
| Key path(s)       | `LoginPath`                  | `CreationPath` + `RegistrationPagePath`|
| Attribute fields  | Username, Password           | + Email, Phone, Address                |
| Rule group name   | `AWSManagedRulesATPRuleSet`  | `AWSManagedRulesACFPRuleSet`           |
| Threat            | Credential stuffing / ATO    | Fake/bulk account creation             |

> **Gotcha:** ATP and ACFP are separate paid groups. Protecting both login and sign-up means adding **both** (each billed separately). One does not cover the other.

---

## CLI

```bash
aws wafv2 describe-managed-rule-group \
  --vendor-name AWS \
  --name AWSManagedRulesACFPRuleSet \
  --scope REGIONAL \
  --region us-east-1
```

Approx WCU: **~50** (plus scope-down). ACFP adds per-registration-request analysis fees (billing).

---

## Gotchas & Caveats

1. **Both paths are required** — `CreationPath` **and** `RegistrationPagePath`. Missing the page path disables navigation/interstitial checks.
2. **Wrong `CreationPath` = no protection**, and it fails silently (same trap as ATP's LoginPath).
3. **`PayloadType` and field pointers must match the real request** — including nested JSON pointers and optional phone/address fields.
4. **Response inspection is CloudFront/ALB only** — no effect on API Gateway/AppSync/Cognito/App Runner.
5. **Same status code for success and failure** breaks StatusCode inspection — use BodyContains/Json.
6. **Token-dependent rules need the SDK** — otherwise legitimate sign-ups look automated.
7. **Billing is per registration request inspected** — a sign-up flood costs; put a rate-based rule + scope-down in front.
8. **ACFP ≠ ATP** — you must deploy both to cover both login and sign-up; each is a separate charge.
9. **Address/phone checks only run if you map those fields** — omitting `AddressFields`/`PhoneNumberFields` reduces fraud-attribute coverage.
10. **Multi-step / progressive registration** (email first, details later) may not fit the single-request model — verify where the actual account-creation POST lands.
11. **Compromised-credential signal reflects the submitted data**, not identity — it flags reuse of breached creds at sign-up.
12. **Version upgrades can shift rule actions** — pin in production, test in Count.

### Official Caveats (from AWS docs)

1. **Not available for Amazon Cognito user pools.** You can't associate a Web ACL that uses ACFP with a Cognito user pool, and you can't add ACFP to a Web ACL already associated with one.
2. **Requires custom configuration of both paths.** You must specify your registration page path and account-creation path. Except where noted, the rules inspect **all** requests sent to those two endpoints.
3. **Inspects request tokens for human-interactivity signals.** ACFP uses request tokens to gather browser info and gauge the level of human interactivity behind each account-creation request.
4. **Aggregates by IP, session, and account attributes.** It detects bulk creation by aggregating on IP address, client session, and provided account data such as physical address and phone number.
5. **Blocks compromised-credential sign-ups.** It detects and blocks new-account creation using credentials known to be compromised.
6. **Intelligent-threat-mitigation rule group with extra fees.** ACFP is billed additional fees; follow intelligent-threat-mitigation best practices to control cost.
7. **Docs describe the latest static version.** Use `DescribeManagedRuleGroup` and the AWS Managed Rules changelog for other versions.

> Source: [AWS WAF Fraud Control account creation fraud prevention (ACFP) rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-acfp.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                    | Cause                                             | Fix                                                          |
| ---------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| ACFP not stopping fake sign-ups          | Wrong CreationPath / payload / fields             | Verify actual POST path & payload; correct config            |
| Navigation checks not working            | RegistrationPagePath missing/wrong                | Set the real form page path                                  |
| Legit sign-ups blocked                   | Volumetric/token rules aggressive or no SDK       | Deploy SDK; override rules to Count, then tune               |
| Response signals ignored                 | Unsupported origin or wrong codes/strings         | Use CloudFront/ALB; match real success/failure signals       |
| No ACFP labels in logs                   | Rule not evaluating (path mismatch)               | Fix CreationPath; confirm requests hit it                    |
| Address/phone fraud not detected         | Fields not mapped                                 | Add AddressFields / PhoneNumberFields identifiers            |
| High ACFP cost                           | Flood on sign-up inspected per request            | Rate-based rule + scope-down in front                        |

---

## Best Practices

1. **Map every relevant field** (email, password, phone, address) for full fraud-attribute coverage.
2. **Set both CreationPath and RegistrationPagePath** to the real endpoints.
3. **Verify the actual sign-up POST** for multi-step flows before configuring.
4. **Enable response inspection on CloudFront/ALB** with the correct signal type.
5. **Deploy in Count first**, watch ACFP labels, then enforce.
6. **Front ACFP with a rate-based rule** scoped to the sign-up path to cap cost.
7. **Integrate the SDK** for token-dependent rules.
8. **Deploy ATP and ACFP together** for full account-lifecycle protection; pin versions and test upgrades.

---

## Useful Links

- [Account Creation Fraud Prevention (ACFP)](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-acfp.html)
- [ACFP Request Inspection](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-acfp.html#aws-managed-rule-groups-acfp-request-inspection)
- [ACFP Response Inspection](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-acfp.html#aws-managed-rule-groups-acfp-response-inspection)
- [ACFP Labels](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-acfp.html#aws-managed-rule-groups-acfp-labels)
- [Client Application Integration (SDKs)](https://docs.aws.amazon.com/waf/latest/developerguide/waf-application-integration.html)
- [Fraud Control Pricing](https://aws.amazon.com/waf/pricing/)

---
