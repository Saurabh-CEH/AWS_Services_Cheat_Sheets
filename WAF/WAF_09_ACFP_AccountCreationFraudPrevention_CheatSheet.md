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

All labels use the prefix `awswaf:managed:aws:acfp:`. Each rule adds a descriptive label **and** a `<RuleName>` label.

| Label (representative)                                                          | Meaning                             |
| ------------------------------------------------------------------------------- | ----------------------------------- |
| `...:signal:credential_compromised`                                             | Stolen credentials at sign-up       |
| `...:signal:missing_credential`                                                 | Sign-up missing credentials (no action) |
| `...:risk_score:high` / `:medium` / `:low`                                      | Risk-score evaluation (only `high` blocks) |
| `...:risk_score:evaluation_failed`                                              | Risk score couldn't be computed     |
| `...:risk_score:contributor:*`                                                  | Per-contributor status (IP reputation, stolen creds, …) |
| `...:signal:automated_browser`                                                  | Automated browser at sign-up        |
| `...:signal:browser_inconsistency`                                              | Inconsistent browser interrogation  |
| `...:signal:client:human_interactivity:low/medium/high` / `:insufficient_data`  | Human-interactivity level           |
| `...:signal:form_detected`                                                      | An HTML form was present            |
| `...:aggregate:volumetric:ip:creation:high/medium/low`                          | Creation rate per IP                |
| `...:aggregate:volumetric:session:creation:high/medium/low`                     | Creation rate per session           |
| `...:aggregate:attribute:username_traversal:creation:high/medium/low`           | Username cycling at sign-up         |
| `...:aggregate:volumetric:phone_number:high/medium/low`                         | Same phone number volume            |
| `...:aggregate:volumetric:address:high/medium/low`                              | Same physical address volume        |
| `...:aggregate:volumetric:ip:successful_creation_response:*` / `failed_creation_response:*` | Creation-response counts per IP (CloudFront) |
| `...:aggregate:volumetric:session:successful_creation_response:*` / `failed_creation_response:*` | Same, per session (CloudFront) |
| `...:aggregate:volumetric:session:creation:token_reuse:ip`                      | One token across >5 IPs             |
| `awswaf:managed:token:absent` / `rejected`                                      | Token state (see CAPTCHA & Challenge sheet) |

```
Priority 30:  ACFP rule group        (adds labels)
Priority 31:  Block → LabelMatch acfp:signal:credential_compromised
Priority 32:  Challenge → LabelMatch acfp:signal:automated_browser
```

---

## Rules Inside the ACFP Group (`AWSManagedRulesACFPRuleSet`, WCU 50)

Complete rule listing (latest static version). **All rules require a token except `UnsupportedCognitoIDP` and `AllRequests`.** **R** = response-inspection rule, **CloudFront only** (not evaluated over HTTP/3 QUIC). Note the mix of **Block**, **CAPTCHA**, and **Challenge** default actions.

| Rule name                          | Action     | Trigger / threshold                                                       | R |
| ---------------------------------- | ---------- | ------------------------------------------------------------------------- | - |
| `UnsupportedCognitoIDP`            | Block      | Traffic to a Cognito user pool (ACFP unsupported there)                   |   |
| `AllRequests`                      | **Challenge** | Every request to the **registration page path** — forces token acquisition **before** other rules run |   |
| `RiskScoreHigh`                    | Block      | Highly suspicious IP/other factors (also emits medium/low + `contributor:` labels; `risk_score:evaluation_failed` if it can't score) |   |
| `SignalCredentialCompromised`      | Block      | Sign-up uses **stolen credentials** (also labels `missing_credential`, no action) |   |
| `SignalClientHumanInteractivityAbsentLow` | **CAPTCHA** | Abnormally **low human interactivity** (mouse/keys/form) — SDK required; creation path only |   |
| `AutomatedBrowser`                 | Block      | Indicators the client browser is **automated**                           |   |
| `BrowserInconsistency`             | **CAPTCHA** | Inconsistent browser interrogation data                                  |   |
| `VolumetricIpHigh`                 | **CAPTCHA** | **>20 creation requests / IP / 10 min** (medium >15, low >10 — no action) |   |
| `VolumetricSessionHigh`            | Block      | **>10 creation requests / session / 30 min** (medium >5, low >1)         |   |
| `AttributeUsernameTraversalHigh`   | Block      | **>10 different usernames / session / 30 min** (medium >5, low >1)       |   |
| `VolumetricPhoneNumberHigh`        | Block      | **>10 sign-ups with same phone / 30 min** (medium >5, low >1)            |   |
| `VolumetricAddressHigh`            | Block      | **>100 sign-ups with same address / 30 min**                             |   |
| `VolumetricAddressLow`             | **CAPTCHA** | Same address: **medium >50**, **low >10** / 30 min                       |   |
| `VolumetricIPSuccessfulResponse`   | Block      | **>10 successful creations / IP / 10 min** (lower threshold than VolumetricIpHigh; from response) | ✓ |
| `VolumetricSessionSuccessfulResponse` | Block   | **>1 successful creation / session / 30 min** (from response; also medium/high + failed labels) | ✓ |
| `VolumetricSessionTokenReuseIp`    | Block      | One token used across **>5 distinct IPs**                                |   |

**Response-inspection rules (R)** inspect up to the first **64 KB** of the response body/JSON and emit medium/low **successful** and **failed** creation-response labels (no action) for custom rules; CloudFront only.

> **Gotcha:** Because almost every ACFP rule needs a **token**, deploy the **JS/Mobile SDK** (or let `AllRequests` Challenge mint one on the registration page). The human-interactivity rule needs the SDK to capture mouse/keyboard/form signals — without it, that rule can't evaluate.

> Rule details rephrased from [AWS WAF Fraud Control ACFP rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-acfp.html) for compliance with licensing restrictions.

---

## Rules & Features Added by Version

ACFP is **versioned** (default vs latest static version can differ — pin in production, verify with `describe-managed-rule-group` and the [changelog](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-changelog.html)):

| Version / date            | Change                                                                                         |
| ------------------------- | ---------------------------------------------------------------------------------------------- |
| **1.1** (2024-09-13)      | Labeling update — every rule now applies a `awswaf:managed:aws:acfp:<RuleName>` label            |
| **1.0** (2024-05-29)      | Rule group became **versioned** (behavior unchanged); default set to 1.0                        |
| 2023-06-13                | **Initial release** of `AWSManagedRulesACFPRuleSet`                                             |

> ACFP is the newest of the three intelligent-threat groups. Unlike Bot Control, ATP/ACFP have not added new *named* rules since launch — changes have been labeling/versioning. Track the changelog for future rule additions.

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
