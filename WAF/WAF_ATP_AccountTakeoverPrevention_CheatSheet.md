# AWS WAF - Account Takeover Prevention (ATP) Cheat Sheet

## Overview

**Account Takeover Prevention (ATP)** is a paid managed rule group (`AWSManagedRulesATPRuleSet`) that protects **login/sign-in** endpoints from credential stuffing, brute force, and use of stolen credentials. It inspects login **requests** (and optionally **responses**) and labels/blocks suspicious sign-in activity.

**Key point:** ATP is login-specific. You must tell it exactly where login happens and how credentials are structured. Misconfiguring the login path or field pointers is the number-one reason ATP "doesn't work."

---

## What ATP Detects

| Capability                          | Description                                                          |
| ----------------------------------- | -------------------------------------------------------------------- |
| **Stolen credential use**           | Checks submitted credentials against AWS's stolen-credential intel   |
| **Credential stuffing / brute force** | High-volume or distributed login attempts                          |
| **Missing / malformed credentials** | Login requests without proper username/password fields               |
| **Login response analysis**         | Reads success/failure from origin to detect guessing (optional)      |
| **Volumetric login patterns**       | Anomalous per-IP / per-session login rates                          |

---

## Required Configuration

ATP needs a `ManagedRuleGroupConfigs` entry describing the login endpoint and payload shape:

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesATPRuleSet",
    "ManagedRuleGroupConfigs": [
      {
        "AWSManagedRulesATPRuleSet": {
          "LoginPath": "/api/login",
          "RequestInspection": {
            "PayloadType": "JSON",
            "UsernameField": { "Identifier": "/email" },
            "PasswordField": { "Identifier": "/password" }
          },
          "ResponseInspection": {
            "StatusCode": {
              "SuccessCodes": [200],
              "FailureCodes": [401, 403]
            }
          },
          "EnableRegexInPath": false
        }
      }
    ]
  }
}
```

| Field                    | Notes                                                                    |
| ------------------------ | ------------------------------------------------------------------------ |
| **LoginPath**            | URI where credentials are POSTed. Matched as a prefix (see gotchas)      |
| **PayloadType**          | `JSON` or `FORM_ENCODED` — must match how the client actually sends creds |
| **UsernameField.Identifier** | JSON pointer (`/email`) or form field name (`username`)              |
| **PasswordField.Identifier** | JSON pointer or form field name                                     |
| **ResponseInspection**   | Optional; reads origin login result. Supported on CloudFront & ALB only  |
| **EnableRegexInPath**    | Allow regex in LoginPath matching                                        |

> **Gotcha:** `PayloadType` must match reality. If your app sends JSON but you configure `FORM_ENCODED` (or vice-versa), ATP can't parse credentials and detection silently degrades — no error is thrown.

---

## Response Inspection (Optional but Powerful)

ATP can read your origin's login **response** to know whether an attempt succeeded, sharpening detection of successful credential-stuffing hits.

| Inspection type | How it identifies success/failure                          |
| --------------- | ---------------------------------------------------------- |
| **StatusCode**  | Lists of success vs failure HTTP codes                     |
| **Header**      | A response header name + success/failure values            |
| **BodyContains**| Success/failure strings in the response body               |
| **Json**        | A JSON field in the response + success/failure values      |

```json
"ResponseInspection": {
  "BodyContains": {
    "SuccessStrings": ["login successful"],
    "FailureStrings": ["invalid credentials"]
  }
}
```

> **Gotcha:** Response inspection works **only on CloudFront and ALB** origins (where WAF can observe the response). API Gateway, AppSync, Cognito, and App Runner do **not** support it — configuring it there has no effect.

> **Gotcha:** If your login endpoint returns `200` for *both* success and failure (common in SPAs that render errors client-side), StatusCode inspection is useless. Use `BodyContains` or `Json` instead.

---

## ATP Labels

ATP stamps labels you can act on with later `LabelMatchStatement` rules:

| Label (representative)                                              | Meaning                          |
| ------------------------------------------------------------------ | -------------------------------- |
| `awswaf:managed:aws:atp:signal:credential_compromised`             | Submitted creds found in breach data |
| `awswaf:managed:aws:atp:signal:missing_credential`                 | Login without proper fields      |
| `awswaf:managed:aws:atp:aggregate:volumetric_session`             | Too many logins per session      |
| `awswaf:managed:aws:atp:aggregate:volumetric_ip_*`                | Too many logins per IP           |
| `awswaf:managed:token:rejected` / `absent`                         | Token state (TARGETED-style)     |

```
Priority 20:  ATP rule group           (adds labels)
Priority 21:  Block → LabelMatch atp:signal:credential_compromised
Priority 22:  CAPTCHA → LabelMatch atp:aggregate:volumetric_session
```

---

## Rules Inside the ATP Group (examples)

| Rule name                        | Action tendency | Purpose                                  |
| -------------------------------- | --------------- | ---------------------------------------- |
| `VolumetricIpHigh`               | Block/Challenge | Excessive logins from one IP             |
| `VolumetricSession`              | Challenge       | Excessive logins in a session            |
| `AttributeCompromisedCredentials`| Block           | Known-stolen credentials submitted       |
| `AttributePasswordTraversal`     | Block           | Password-field manipulation              |
| `AttributeUsernameTraversal`     | Block           | Username-field manipulation              |
| `MissingCredential`              | Block/Count     | Malformed login attempt                  |

> **Gotcha:** Some ATP rules depend on the WAF **token** (like Bot Control TARGETED). Real users need the JS/Mobile SDK, or token-based rules treat them as suspicious.

---

## Scope & Placement

- Add ATP **once per login path**. Multiple distinct login endpoints may need multiple ATP rule statements (each with its own `LoginPath`).
- Place ATP **before** your label-match enforcement rules (lower priority number).
- Combine with a **rate-based rule** scoped to the login path for cheap volumetric backstop.

---

## CLI

```bash
# Inspect ATP group rules, labels, WCU
aws wafv2 describe-managed-rule-group \
  --vendor-name AWS \
  --name AWSManagedRulesATPRuleSet \
  --scope REGIONAL \
  --region us-east-1
```

Approx WCU: **~50** (plus scope-down). ATP adds per-login-request analysis fees (billing).

---

## Gotchas & Caveats

1. **Wrong `LoginPath` = zero protection.** ATP only inspects requests to the configured path. A trailing-slash mismatch, wrong prefix, or a login served under a different route means ATP never runs — and it fails silently.
2. **`PayloadType` mismatch breaks credential parsing** — JSON vs FORM_ENCODED must match the actual request.
3. **Field identifiers must match the payload exactly** — `/email` vs `/username`, nested pointers (`/user/email`), or form field names. A wrong pointer = ATP can't read credentials.
4. **Response inspection is CloudFront/ALB only** — configuring it on API Gateway/AppSync/Cognito/App Runner does nothing.
5. **Same status code for success and failure defeats StatusCode inspection** — use BodyContains/Json/Header for SPA-style logins.
6. **ATP inspects only the login endpoint, not the whole site** — it is not a general bot/DDoS control. Pair with Bot Control and rate-based rules.
7. **Token-dependent rules need the SDK** — without it, real users can look volumetric/suspicious.
8. **Billing is per login request inspected** — a login page hit by a flood still costs; scope-down and rate-limit in front to cap cost.
9. **Cognito hosted UI** login flows have their own structure — verify the actual POST path/payload before configuring; the visible URL is not always the credential-submit endpoint.
10. **Credential-compromised detection acts on the submitted credentials**, not the user's identity — rotating a breached password fixes the label; blocking the IP does not.
11. **Multiple login endpoints need multiple ATP statements** — one `LoginPath` per statement.
12. **Version upgrades can change rule actions** — pin the version in production, test in Count.

### Official Caveats (from AWS docs)

1. **Not available for Amazon Cognito user pools.** You can't associate a Web ACL that uses ATP with a Cognito user pool, and you can't add ATP to a Web ACL already associated with one.
2. **Response inspection is for CloudFront distributions.** ATP inspects your application's login responses (to track success/failure and temporarily block clients with too many failures) on **CloudFront**. Response inspection runs **asynchronously**, so it does not add latency to your traffic.
3. **Requires specific configuration.** ATP must be configured with your login endpoint and credential field details; it won't work out of the box.
4. **It's an intelligent-threat-mitigation rule group with extra fees.** You're charged additional fees for ATP; follow the intelligent-threat-mitigation best practices to control cost.
5. **Stolen-credential checks use a regularly updated database.** ATP checks email/password combinations against a stolen-credential database that is updated as new leaked credentials are found; it aggregates by IP address and client session.
6. **Docs describe the latest static version.** Behavior can differ by version — use `DescribeManagedRuleGroup` and the AWS Managed Rules changelog for other versions.

> Source: [AWS WAF Fraud Control account takeover prevention (ATP) rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-atp.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                     | Cause                                               | Fix                                                          |
| ----------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| ATP not detecting credential stuffing     | Wrong LoginPath / field identifiers / payload type  | Verify actual POST path + payload; correct pointers & type   |
| Response signals ignored                  | Origin not CloudFront/ALB, or wrong codes/strings   | Use supported origin; match real success/failure signals     |
| Legit users blocked at login              | Volumetric/token rules too aggressive or no SDK     | Deploy SDK; override volumetric rules to Count, then tune     |
| No ATP labels in logs                     | ATP rule not evaluating (path mismatch)             | Fix LoginPath; confirm requests hit it                       |
| SPA login always "success" to ATP         | 200 for both outcomes                                | Switch to BodyContains/Json response inspection              |
| High ATP cost                             | Flood hitting login inspected per request           | Rate-based rule + scope-down in front of ATP                 |

---

## Best Practices

1. **Confirm the real login POST path and payload** (via browser dev tools / API contract) before configuring — don't assume the URL.
2. **Match PayloadType and field identifiers exactly** to the request.
3. **Enable response inspection on CloudFront/ALB** for sharper detection; pick the right signal type for your login response.
4. **Deploy in Count first**, watch ATP labels, then enforce with label-match rules.
5. **Front ATP with a rate-based rule** scoped to the login path to cap volumetric cost and add defense-in-depth.
6. **Integrate the SDK** if you enable token-dependent rules.
7. **Handle compromised-credential events** by forcing password resets, not just blocking IPs.
8. **Pin the managed group version** and test upgrades before adopting.

---

## Useful Links

- [Account Takeover Prevention (ATP)](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-atp.html)
- [ATP Request Inspection](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-atp.html#aws-managed-rule-groups-atp-request-inspection)
- [ATP Response Inspection](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-atp.html#aws-managed-rule-groups-atp-response-inspection)
- [ATP Labels](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-atp.html#aws-managed-rule-groups-atp-labels)
- [Client Application Integration (SDKs)](https://docs.aws.amazon.com/waf/latest/developerguide/waf-application-integration.html)
- [Fraud Control Pricing](https://aws.amazon.com/waf/pricing/)

---
