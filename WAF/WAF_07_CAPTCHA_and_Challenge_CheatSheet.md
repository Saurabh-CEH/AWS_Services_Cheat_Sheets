# AWS WAF - CAPTCHA & Challenge Actions Cheat Sheet

## Overview

**CAPTCHA** and **Challenge** are WAF rule **actions** (like Allow/Block/Count) that use **AWS WAF token management** to verify a client is a real browser/human before letting the request through. They are part of WAF's **intelligent threat mitigation** and are used both as explicit rule actions and internally by Bot Control, ATP/ACFP, and the Anti-DDoS rule group.

**Key point:** Both actions decide what to do based on the **state of the request's WAF token** and your **immunity time** configuration. A request with a **valid token** is treated like **Count** (evaluation continues); a request with a **missing/invalid/expired token** is **blocked** and sent an interstitial to solve.

---

## CAPTCHA vs Challenge

| Aspect                  | Challenge                                          | CAPTCHA                                              |
| ----------------------- | -------------------------------------------------- | ---------------------------------------------------- |
| Who solves it           | The **browser** (silent, no user interaction)      | The **end user** (visual/interactive puzzle)         |
| User friction           | None (runs silently)                               | High (user must solve a puzzle)                      |
| Response status code    | **202** Accepted                                   | **405** Method Not Allowed                           |
| Response header         | `x-amzn-waf-action: challenge`                     | `x-amzn-waf-action: captcha`                         |
| Interstitial (if `Accept: text/html`) | JavaScript **challenge script**    | JavaScript **CAPTCHA puzzle** (runs challenge first) |
| Proves                  | Client is a real browser environment               | Client is a real browser **and** likely human        |
| Use when                | Block automation with minimal UX impact            | High-value/abused endpoints where you'll accept friction |

> A CAPTCHA interstitial **runs the challenge first** (to establish the browser + token), **then** shows the puzzle. So CAPTCHA is a superset of Challenge.

---

## How the Action Handles a Request (token state machine)

```
Request matches a rule with CAPTCHA/Challenge action
        │
        ▼
  Inspect WAF token
        │
   ┌────┴─────────────────────────────┐
   │                                  │
Valid token                    Missing / invalid / expired token
(unexpired solve,              │
 valid domain)                 ▼
   │                    STOP web ACL evaluation → BLOCK the request
   ▼                    Send interstitial response:
Treated like COUNT:         Challenge → HTTP 202 + x-amzn-waf-action: challenge
 - apply labels/customizations   CAPTCHA  → HTTP 405 + x-amzn-waf-action: captcha
 - CONTINUE to next rules        (+ JS interstitial if Accept: text/html)
                              │
                              ▼
                    Client solves silently (Challenge) or user solves puzzle (CAPTCHA)
                    → interstitial updates token with solve timestamp
                    → resubmits the ORIGINAL request with the updated token
```

- **Valid token** = present, contains a valid **challenge or CAPTCHA solution**, an **unexpired** timestamp, and a **domain valid** for the Web ACL.
- After a successful solve, the script **initializes a token** (if none) and **resubmits** the original request automatically.

---

## Token Status Labels

WAF adds labels reflecting token state (visible to later rules + CloudWatch metrics):

| Label                                          | Meaning                                                        |
| ---------------------------------------------- | -------------------------------------------------------------- |
| `awswaf:managed:token:accepted`                | Valid challenge solution, unexpired, valid domain              |
| `awswaf:managed:captcha:accepted`              | Valid CAPTCHA solution                                         |
| `...:token:rejected:not_solved`                | Token missing the challenge/CAPTCHA solution                   |
| `...:token:rejected:expired`                   | Solve timestamp exceeded the configured **immunity time**      |
| `...:token:rejected:domain_mismatch`           | Token domain not in the Web ACL's **token domain** config      |
| `...:token:rejected:invalid`                   | Token couldn't be read                                         |
| `...:token:absent` / `...:captcha:absent`      | No token on the request                                        |
| `awswaf:managed:token:id:<id>`                 | Client-session identifier (changes on new token)              |
| `awswaf:managed:token:fingerprint:<id>`        | Browser fingerprint (stable across token attempts; not unique) |

---

## Immunity Time (token expiration)

**Immunity time** = how long a solved token is honored before the client must solve again. Configurable at the **Web ACL** level and overridable per **rule**:

| Setting                         | Controls                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| **Challenge immunity time**     | How long a solved **challenge** is valid                       |
| **CAPTCHA immunity time**       | How long a solved **CAPTCHA** is valid                         |
| **Token immunity (timestamp)**  | Applied when the token is checked; expiry → `rejected:expired` |

- Shorter immunity = more re-challenges (more friction/cost, higher assurance). Longer = smoother UX, weaker.

---

## Token Domains

By default, WAF accepts a token **only for the domain of the associated resource**. To share tokens across hosts/subdomains/APIs, configure a **token domain list** on the Web ACL:

- With a token domain list, WAF accepts tokens for **all listed domains** plus the associated resource's domain.
- **Misconfigured token domains → `rejected:domain_mismatch`** → every cross-host request looks unsolved and gets challenged in a loop.

---

## CORS / Header Visibility Caveat

- CAPTCHA/Challenge responses **do not include CORS headers**.
- JavaScript apps running in the browser therefore **cannot read** the `x-amzn-waf-action` header **cross-domain** — it's only readable within the application's own domain.
- For SPAs/cross-origin API calls, use the **WAF JavaScript/Mobile SDK** to acquire and manage tokens instead of relying on reading the response header.

---

## Acquiring Tokens: SDK vs Rule Actions

| Method                        | Pros                                              | Cons                                             |
| ----------------------------- | ------------------------------------------------- | ------------------------------------------------ |
| **Application integration SDK** (JS / iOS / Android) | Silent, best UX, works for APIs/SPAs/native | Requires code integration                        |
| **CAPTCHA / Challenge rule actions** | No app code — just add the action           | Added cost, more UX intrusion, **requires JavaScript/HTML client** |

---

## CLI / Config Example

Rule using the Challenge action:

```json
{
  "Name": "challenge-login",
  "Priority": 5,
  "Statement": { "ByteMatchStatement": {
    "SearchString": "/login",
    "FieldToMatch": { "UriPath": {} },
    "PositionalConstraint": "STARTS_WITH",
    "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
  }},
  "Action": { "Challenge": {} },
  "VisibilityConfig": { "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "challengeLogin" }
}
```

Set token domains + immunity times on the Web ACL (excerpt):

```json
{
  "TokenDomains": ["example.com", "api.example.com"],
  "ChallengeConfig": { "ImmunityTimeProperty": { "ImmunityTime": 300 } },
  "CaptchaConfig":   { "ImmunityTimeProperty": { "ImmunityTime": 300 } }
}
```

---

## Pricing

- Using **CAPTCHA or Challenge** (as a rule action **or** as a rule-action override in a rule group) incurs **additional per-attempt fees** on top of base WAF request charges.
- Bot Control / Anti-DDoS rules that issue Challenge/CAPTCHA also trigger these fees. Monitor `CaptchaRequests` / `ChallengeRequests` metrics.

---

## Gotchas & Caveats

1. **Valid token = Count, not Allow** — the request keeps being evaluated by later rules; a valid CAPTCHA solve does **not** short-circuit to allow.
2. **Non-browser/API clients can't solve** — pure API, server-to-server, and older native clients fail the interstitial. Use the SDK or scope-down/exempt those paths.
3. **CORS headers absent** — browser JS can't read `x-amzn-waf-action` cross-domain; don't build client logic around reading it cross-origin.
4. **Token domain mismatch loops** — cross-subdomain/API setups without a token domain list see every request as token-absent.
5. **Immunity time trade-off** — too short = constant re-challenges (cost + friction); too long = weak protection.
6. **CAPTCHA runs a challenge first** — a client that can't run the challenge script never even reaches the puzzle.
7. **Billed per attempt** — heavy challenge/CAPTCHA volume (e.g., during a DDoS event via Anti-DDoS) adds real cost.
8. **`Accept: text/html` needed for the interstitial** — non-HTML clients get the status code (202/405) but no solvable page.

---

## Troubleshooting

| Issue                                        | Cause                                            | Fix                                                       |
| -------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| Legit users stuck in challenge loop          | Token domain mismatch / can't run JS             | Configure token domains; integrate SDK; exempt API paths  |
| API clients getting 202/405                  | Non-browser client can't solve interstitial       | Scope-down the rule; use SDK tokens for API callers       |
| Users re-challenged constantly               | Immunity time too short                          | Increase Challenge/CAPTCHA immunity time                  |
| SPA can't read `x-amzn-waf-action`           | No CORS headers on CAPTCHA/Challenge responses    | Use the JavaScript SDK instead of reading the header      |
| Unexpected cost spike                        | High challenge/CAPTCHA volume (attack or bots)   | Review `CaptchaRequests`/`ChallengeRequests`; tune sensitivity/scope |

---

## Best Practices

1. **Prefer Challenge over CAPTCHA** unless you need human proof — Challenge is silent (no UX hit).
2. **Integrate the SDK** for SPAs, APIs, and native apps so tokens are acquired silently.
3. **Configure token domains** for every subdomain/API host sharing the protection.
4. **Tune immunity time** to balance friction, cost, and assurance.
5. **Scope-down** CAPTCHA/Challenge to sensitive endpoints (login, checkout, signup), not static assets.
6. **Monitor `CaptchaRequests`/`ChallengeRequests`** metrics for cost and false-positive signals.
7. **Test in Count / with a label-match rule** before broadly enforcing challenge actions.

---

## Useful Links

- [CAPTCHA and Challenge action behavior](https://docs.aws.amazon.com/waf/latest/developerguide/waf-captcha-and-challenge-actions.html)
- [Using CAPTCHA and Challenge](https://docs.aws.amazon.com/waf/latest/developerguide/waf-captcha-and-challenge.html)
- [Token domains](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-captcha-challenge-token-domains.html)
- [Token immunity times](https://docs.aws.amazon.com/waf/latest/developerguide/waf-tokens-immunity-times.html)
- [Token use in intelligent threat mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/waf-tokens.html)
- [Application integration SDKs](https://docs.aws.amazon.com/waf/latest/developerguide/waf-application-integration.html)
- [AWS WAF Pricing](https://aws.amazon.com/waf/pricing/)

---

*Sources: AWS official documentation. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
