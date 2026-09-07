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

## Token Labels

WAF token management adds labels reflecting token state, visible to rules that run **after** the labeling rule group and recorded to CloudWatch label metrics.

> **Important:** These token labels are applied **only when you use an intelligent-threat-mitigation managed rule group** — **Bot Control, ATP, ACFP, or Anti-DDoS**. Plain CAPTCHA/Challenge **rule actions by themselves do not add these labels**. So to label/route on token state, run one of those managed groups (even in Count) ahead of your label-match rules.

### 1. Informational labels (no CloudWatch metrics for these)

| Label                                            | Meaning                                                                 |
| ------------------------------------------------ | ----------------------------------------------------------------------- |
| `awswaf:managed:token:id:<identifier>`           | Unique **client-session** identifier; **changes** if the client acquires a new token (e.g. after discarding one) |
| `awswaf:managed:token:fingerprint:<identifier>`  | Robust **browser fingerprint** from client signals; **stable** across token-acquisition attempts, and **not unique** to a single client |

### 2. Token status labels — two namespace prefixes

Status labels always start with one of these prefixes, then a status name:

| Prefix                       | Reports on                                                        |
| ---------------------------- | ----------------------------------------------------------------- |
| `awswaf:managed:token:`      | General token status **and** the token's **challenge** information |
| `awswaf:managed:captcha:`    | The token's **CAPTCHA** information                                |

### 3. Status names (apply under either prefix)

| Status name                     | Meaning                                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------------------------ |
| `accepted`                      | Token present with a **valid** challenge/CAPTCHA solution, an **unexpired** timestamp, and a **valid domain** |
| `rejected`                      | Token present but fails acceptance — always paired with one reason below                          |
| `rejected:not_solved`           | Token is **missing** the challenge/CAPTCHA solution                                              |
| `rejected:expired`              | Challenge/CAPTCHA timestamp **expired** per the Web ACL's configured immunity time               |
| `rejected:domain_mismatch`      | Token domain **not a match** for the Web ACL's token-domain configuration                        |
| `rejected:invalid`              | WAF **couldn't read** the token                                                                  |
| `absent`                        | Request has **no token** (or the token manager couldn't read it)                                 |

**Combining prefix + status** gives the full label. Examples:
- `awswaf:managed:token:accepted` — valid **challenge** solution, unexpired, valid domain.
- `awswaf:managed:captcha:accepted` — valid **CAPTCHA** solution.
- `awswaf:managed:captcha:rejected` **+** `awswaf:managed:captcha:rejected:expired` — the CAPTCHA timestamp exceeded the CAPTCHA immunity time.
- `awswaf:managed:token:absent` / `awswaf:managed:captcha:absent` — no token on the request.

> A `rejected` label is always accompanied by its reason label (the `rejected:*` variant), so you'll see two labels together.

### Why a token / CAPTCHA is `absent`

`awswaf:managed:token:absent` (or `captcha:absent`) means the request arrived **with no WAF token at all** (or the token manager couldn't read one). Common scenarios that cause this:

| Scenario                                             | Why it produces `absent`                                                        |
| ---------------------------------------------------- | ------------------------------------------------------------------------------- |
| **First request in a session**                       | The client hasn't been challenged yet, so no token has been issued              |
| **No SDK integration + never challenged**            | Without the JS/Mobile SDK, a token is only minted after a Challenge/CAPTCHA solve; if nothing has challenged the client, it stays token-less |
| **Non-browser / API / server-to-server client**      | CLIs, backend calls, IoT, and old native apps don't run the JS interstitial or carry the cookie/header, so they never acquire a token |
| **Client strips or doesn't send the token**          | Cookies disabled, privacy tooling, or a client that drops the `aws-waf-token` cookie / header |
| **Cross-domain / subdomain call without token domains** | The token exists but for a different host; the token manager sees none valid for this domain (borderline with `domain_mismatch`, but a wholly missing token reads as `absent`) |
| **Token expired and discarded**                      | After immunity time passes, a client may discard the old token and send the next request with none |
| **CDN / proxy strips the cookie or header**          | An intermediary (custom CDN, corporate proxy) removes the `aws-waf-token` cookie or `x-aws-waf-token` header before it reaches WAF |
| **New `token:id`**                                   | When a client acquires a fresh token the session id changes; the request just before re-acquisition can appear token-less |
| **Automated/bot traffic**                            | Bots typically don't solve challenges or persist tokens — a flood of `absent` labels on protected paths is itself a bot signal |

**CAPTCHA-specific:** `captcha:absent` (or a token with only a challenge solve, not a CAPTCHA solve) happens when the client has a token from a **silent Challenge** but has **never solved a CAPTCHA puzzle** — so any rule needing a CAPTCHA solve treats it as absent until the puzzle is completed.

> **Takeaway:** `absent` is normal for the very first request and for legitimate non-browser clients. It's a problem only when you *expect* a token (e.g., a browser that should have run the SDK) — then look at SDK integration, cookie/header stripping by a CDN/proxy, and token-domain configuration.

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
9. **Token labels need a managed group** — `awswaf:managed:token:*` / `awswaf:managed:captcha:*` labels are added **only** by Bot Control / ATP / ACFP / Anti-DDoS. If you want to match on token state, run one of those (even in Count) before your label-match rules; plain CAPTCHA/Challenge actions won't emit them.
10. **`token:id` changes on new tokens** — don't use it as a stable client key across sessions; the **fingerprint** label is more stable but **not unique** per client.
11. **`rejected` comes with a reason label** — filter on the specific `rejected:*` (expired / domain_mismatch / not_solved / invalid) to diagnose, not just `rejected`.

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
- [Types of token labels](https://docs.aws.amazon.com/waf/latest/developerguide/waf-tokens-labeling.html)
- [Application integration SDKs](https://docs.aws.amazon.com/waf/latest/developerguide/waf-application-integration.html)
- [AWS WAF Pricing](https://aws.amazon.com/waf/pricing/)

---

*Sources: AWS official documentation. Content was rephrased for compliance with licensing restrictions. Always refer to the official AWS documentation for the most current information.*
