# AWS WAF - Bot Control Cheat Sheet

## Overview

**AWS WAF Bot Control** is a paid managed rule group (`AWSManagedRulesBotControlRuleSet`) that detects and categorizes automated (bot) traffic. It labels requests by bot category and verification status; you then decide what to allow, block, count, or challenge.

**Key point:** Bot Control does **not** block by default across the board — most of its detection rules are designed to be paired with **labels** and your own label-match rules, or run at their built-in actions. You control cost and behavior with **inspection level**, **scope-down**, and **rule action overrides**.

---

## Inspection Levels

Bot Control has two levels, set in `ManagedRuleGroupConfigs`:

| Level        | Detects                                                                 | Relative cost | Needs SDK? |
| ------------ | ----------------------------------------------------------------------- | ------------- | ---------- |
| **COMMON**   | Self-identifying bots, common scrapers, basic automation, verified crawlers | Lower     | No         |
| **TARGETED** | Sophisticated/evasive bots via browser fingerprinting, challenge, CAPTCHA, ML, and behavioral signals | Higher | Recommended (token) |

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesBotControlRuleSet",
    "ManagedRuleGroupConfigs": [
      { "AWSManagedRulesBotControlRuleSet": { "InspectionLevel": "TARGETED" } }
    ]
  }
}
```

> **Gotcha:** TARGETED includes rules that issue silent **Challenge** and **CAPTCHA** actions and rely on a WAF **token**. Without the JavaScript/Mobile SDK integration, legitimate clients that can't run the challenge (APIs, some SPAs, native apps) may get blocked or loop.

---

## Rule Categories Inside the Group

### COMMON-level rules (examples)

Most COMMON category/signal rules apply their action **only to unverified bots** — verified bots are labeled but not actioned. The exception is `CategoryAI`, which applies to all matches.

| Rule name                          | Action (unverified) | Purpose                                                     |
| ---------------------------------- | ------------------- | ----------------------------------------------------------- |
| `CategoryHttpLibrary`              | Block               | Requests from HTTP client libraries (may be legit APIs)     |
| `CategoryScrapingFramework`        | Block               | Web-scraping frameworks                                     |
| `CategorySearchEngine`             | Block               | Search-engine crawlers                                      |
| `CategorySocialMedia`              | Block               | Social-media content-summary bots                           |
| `CategoryAI`                       | Block (all matches) | AI bots — **applies to verified and unverified alike**      |
| `CategoryContentFetcher`           | Block               | Fetches content on behalf of a user (RSS, validation)       |
| `CategorySecurity` / `CategorySeo` / `CategoryMonitoring` / `CategoryArchiver` / `CategoryLinkChecker` / `CategoryPagePreview` / `CategoryEmailClient` / `CategoryWebhooks` / `CategoryAdvertising` / `CategoryMiscellaneous` | Block | Other bot categories |
| `SignalNonBrowserUserAgent`        | Block               | UA doesn't look like a browser (can include API requests)   |
| `SignalKnownBotDataCenter`         | Block               | Source is a known bot data-center range                     |
| `SignalAutomatedBrowser`           | Block               | Indicators the client browser is automated                  |

> For a verified bot, category/signal rules take no action but still add `awswaf:managed:aws:bot-control:bot:verified` plus the bot name/category labels — so you can Allow verified bots ahead of your blocks.

### TARGETED-level rules (accurate actions & thresholds)

All `TGT_*` rules apply only to **unverified** bots. Most rely on a **token** (added by the SDKs or by the CAPTCHA/Challenge actions). Documented thresholds can vary slightly due to latency.

| Rule name                         | Default action | Trigger / notes                                                                 |
| --------------------------------- | -------------- | ------------------------------------------------------------------------------- |
| `TGT_TokenAbsent`                 | **Count**      | Request has no valid challenge token                                            |
| `TGT_VolumetricIpTokenAbsent`     | **Challenge**  | 5+ requests from one client IP in 5 min missing a valid token                   |
| `TGT_VolumetricSession`           | **CAPTCHA**    | Abnormally high requests in one session (5-min window); **needs a token**; can take **5 min** to take effect after enabling |
| `TGT_VolumetricSessionMaximum`    | **Block**      | Same, at maximum confidence; needs a token                                      |
| `TGT_SignalAutomatedBrowser`      | **CAPTCHA**    | Token indicates an automated browser; needs a token                             |
| `TGT_SignalBrowserAutomationExtension` | **CAPTCHA** | Automation extension present (e.g., Selenium IDE); needs a token               |
| `TGT_SignalBrowserInconsistency`  | **CAPTCHA**    | Inconsistent browser interrogation data; needs a token                          |
| `TGT_ML_CoordinatedActivityLow/Medium/High` | Challenge / CAPTCHA / CAPTCHA | ML coordinated-activity detection; can take **up to 24 h** to take effect after enabling the ML option |
| `TGT_TokenReuseIp Low/Medium/High` | Count / CAPTCHA / Block | One token reused across >2 / >5 / >8 distinct IPs in 5 min             |
| `TGT_TokenReuseCountry Low/Medium/High` | Count / CAPTCHA / Block | One token reused across >1 / >2 / >3 countries in 5 min            |
| `TGT_TokenReuseAsn Low/Medium/High` | Count / CAPTCHA / Block | One token reused across >1 / >2 / >3 ASNs in 5 min                    |

> **Gotcha:** `TGT_TokenAbsent` defaults to **Count** (not Block) — enabling TARGETED alone does not block token-less requests via this rule. `TGT_VolumetricIpTokenAbsent` (Challenge) is what actually acts on token-absent floods. Without the SDK, override the token-dependent rules to Count while you roll out.

> Rule details rephrased from [AWS WAF Bot Control rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html) for compliance with licensing restrictions.

---

## Bot Labels

Bot Control adds labels you can act on with `LabelMatchStatement` in a **later** (higher priority-number) rule.

| Label (representative)                                            | Meaning                        |
| ---------------------------------------------------------------- | ------------------------------ |
| `awswaf:managed:aws:bot-control:bot:category:search_engine`      | Search-engine crawler          |
| `awswaf:managed:aws:bot-control:bot:category:social_media`       | Social-media crawler           |
| `awswaf:managed:aws:bot-control:bot:category:scraping_framework` | Scraping tool                  |
| `awswaf:managed:aws:bot-control:bot:category:http_library`       | HTTP client library            |
| `awswaf:managed:aws:bot-control:bot:name:googlebot`              | Specific bot name              |
| `awswaf:managed:aws:bot-control:bot:verified:true`               | Verified good bot              |
| `awswaf:managed:aws:bot-control:signal:automated_browser`        | Automated browser signal       |
| `awswaf:managed:aws:bot-control:signal:non_browser_user_agent`   | Non-browser UA                 |
| `awswaf:managed:token:rejected` / `absent`                        | Token state                    |

> **Gotcha:** Verified-bot labels (`verified:true`) exist so you can **allow legitimate crawlers** (Googlebot, Bingbot). If you blanket-block bot categories without allowing verified bots first, you'll deindex your site from search engines.

---

## Recommended Rule Ordering (Label-Driven)

```
Priority 10:  BotControl rule group           (OverrideAction: None — or Count during rollout)
                 → adds labels, may issue Challenge/CAPTCHA for TARGETED
Priority 11:  Allow  → LabelMatch bot:verified:true         (let good crawlers in)
Priority 12:  Allow  → LabelMatch bot:name:googlebot        (belt-and-suspenders)
Priority 13:  Block  → LabelMatch bot:category:scraping_framework
Priority 14:  Challenge → LabelMatch signal:automated_browser
Default action: ALLOW
```

- The group runs first and **stamps labels**.
- Your label rules run **after** and make the allow/block decision.
- Because labels are only visible to rules with a **higher priority number**, the group must have a lower number than the label-match rules.

---

## Scope-Down (Cost & Precision)

Bot Control is billed per inspected request. Restrict it to the paths that matter:

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesBotControlRuleSet",
    "ManagedRuleGroupConfigs": [
      { "AWSManagedRulesBotControlRuleSet": { "InspectionLevel": "COMMON" } }
    ],
    "ScopeDownStatement": {
      "NotStatement": {
        "Statement": {
          "ByteMatchStatement": {
            "SearchString": "/static/",
            "FieldToMatch": { "UriPath": {} },
            "PositionalConstraint": "STARTS_WITH",
            "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
          }
        }
      }
    }
  }
}
```

> **Gotcha:** Without a scope-down, Bot Control inspects **every** request to the Web ACL — including static assets, health checks, and images — inflating the bill. Exclude non-application paths.

---

## Application Integration (Token / SDK)

TARGETED features depend on a WAF **token** (a cookie/header the SDK manages).

| Integration        | Use                                                              |
| ------------------ | ---------------------------------------------------------------- |
| **JavaScript SDK** | Browsers — issues & refreshes the token, runs silent challenges  |
| **Mobile SDK (iOS/Android)** | Native apps — token acquisition for app traffic        |
| **Token domains**  | Configure which domains the token is valid for (subdomains/APIs) |

> **Gotcha:** The token is only honored on requests to the **same domain scope** you configured. Cross-subdomain or API-on-different-host setups need explicit **token domain** configuration, or every request looks token-absent.

> **Gotcha:** TARGETED challenge/CAPTCHA needs the client to execute JavaScript. Pure API clients, server-to-server calls, and older mobile apps can't — plan to exempt those paths or run them at Count.

---

## Common Configuration Patterns

### Allow good bots, block scrapers, challenge unknown automation

```
BotControl (COMMON)               → labels
Allow  LabelMatch verified:true
Block  LabelMatch category:scraping_framework
Block  LabelMatch category:http_library     (careful: may catch legit integrations)
Challenge LabelMatch signal:automated_browser
```

### Protect a checkout/search endpoint with TARGETED

```
BotControl (TARGETED) scope-down to /search, /checkout
Override TGT_TokenAbsent → Count until SDK is deployed
After SDK rollout: TGT_TokenAbsent → Challenge/Block
```

---

## CLI

```bash
# Inspect the rules, labels, and WCU of the Bot Control group
aws wafv2 describe-managed-rule-group \
  --vendor-name AWS \
  --name AWSManagedRulesBotControlRuleSet \
  --scope REGIONAL \
  --region us-east-1
```

Approx WCU: **~50** (plus scope-down cost). TARGETED processing adds analysis cost per request (billing), not necessarily WCU.

---

## Gotchas & Caveats

1. **Blanket-blocking bot categories deindexes you** — always Allow `verified:true` before blocking categories, or search engines can't crawl.
2. **`category:http_library` catches legitimate integrations** — many partner/API integrations use standard HTTP libraries. Blocking this category can break B2B traffic; prefer Count + allowlist.
3. **TARGETED without the SDK causes challenge loops or false blocks** — token-dependent rules see legitimate non-JS clients as token-absent.
4. **Token domain mismatch = everything looks token-absent** — configure token domains for all subdomains/API hosts sharing the protection.
5. **No scope-down = you pay to inspect static assets** — Bot Control is per-request priced; exclude images/CSS/JS/health checks.
6. **Bot Control labels only reach later rules** — the group's priority number must be **lower** than your label-match rules, or the labels aren't visible.
7. **COMMON won't catch sophisticated bots** — evasive/residential-proxy bots need TARGETED; don't assume COMMON is enough for scraping-heavy targets.
8. **Enabling the group doesn't block anything by itself for many signals** — you must add label-match rules (or verify which internal rules block by default) to get enforcement.
9. **CAPTCHA/Challenge are billed** — heavy challenge volume adds cost; monitor `CaptchaRequests`/`ChallengeRequests`.
10. **Behind CloudFront/ALB with proxies**, ensure the client IP/token headers survive; otherwise volumetric/IP-based `TGT_*` rules mis-aggregate.
11. **FMS-deployed Bot Control** can appear "missing" in member accounts if the policy scope/remediation is off — check the Firewall Manager policy, not just the account's Web ACL.
12. **Version changes** to the managed group can shift rule actions — pin a version in production and test upgrades in Count.

### Official Caveats (from AWS docs)

1. **`TGT_VolumetricSession` needs time to warm up** — it compares current traffic to baselines AWS WAF computes, and can take about **5 minutes** to go into effect after you enable it.
2. **ML coordinated-activity rules take up to 24 hours** — `TGT_ML_CoordinatedActivity*` rules can take **up to 24 h** after enabling the TARGETED rules with the ML option, because baselines are only computed while those rules are in use. A match or two may be a false positive; many matches indicate a real coordinated attack.
3. **Token-dependent rules only apply when a token is present** — `TGT_VolumetricSession`, `TGT_Signal*`, and token-reuse rules only evaluate requests that carry a token (added by the SDKs or by CAPTCHA/Challenge actions).
4. **Thresholds can drift slightly due to latency** — a few requests may get through beyond a documented limit before the rule action applies.
5. **Category/signal rules act on unverified bots only** — verified bots are labeled but not actioned, except `CategoryAI`, which applies to all matches.
6. **ML models are updated periodically** — sudden changes in bot predictions can result; AWS advises contacting Support/your account manager if predictions shift substantially.

> Source: [AWS WAF Bot Control rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                    | Cause                                              | Fix                                                        |
| ---------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------- |
| Googlebot / crawlers blocked             | Blocking categories without allowing verified bots | Add Allow on `bot:verified:true` before category blocks    |
| Real users hit repeated CAPTCHA/Challenge | TARGETED token-dependent rules, no SDK             | Integrate JS/Mobile SDK; configure token domains           |
| Partner API traffic blocked              | `category:http_library` enforced                   | Override that rule to Count; allowlist partner IPs/keys     |
| Higher-than-expected bill                | Inspecting all traffic incl. static                | Add scope-down to app paths only                           |
| Group "does nothing"                     | No label-match rules added / OverrideAction=Count  | Add label rules; set OverrideAction=None                   |
| `token:absent` on legit browsers         | Token domain misconfigured / SDK not loaded        | Fix token domains; verify SDK script loads first           |
| FMS Bot Control not in member accounts   | Policy scope/remediation                           | Review FMS policy scope and enable remediation             |

---

## Best Practices

1. **Roll out in Count first** (OverrideAction=Count), watch labels in logs, then enforce with label rules.
2. **Always Allow verified bots** ahead of category blocks.
3. **Deploy the SDK before enforcing TARGETED** token rules; exempt non-JS paths.
4. **Scope down** to real application endpoints to control cost.
5. **Prefer Challenge over Block** for suspected automation so real browsers pass silently.
6. **Treat `http_library` carefully** — it frequently catches legitimate automation.
7. **Pin the managed group version** in production; test upgrades in a staging Web ACL.
8. **Monitor CAPTCHA/Challenge metrics** to catch loops and cost spikes early.

---

## Useful Links

- [AWS WAF Bot Control](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html)
- [Bot Control Inspection Levels](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html#aws-managed-rule-groups-bot-inspection-levels)
- [Bot Control Rules & Labels](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html#aws-managed-rule-groups-bot-rules)
- [CAPTCHA and Challenge Actions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-captcha-and-challenge.html)
- [Client Application Integration (SDKs)](https://docs.aws.amazon.com/waf/latest/developerguide/waf-application-integration.html)
- [Token Domains](https://docs.aws.amazon.com/waf/latest/developerguide/waf-tokens-domains.html)
- [Bot Control Example Configurations](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot-examples.html)

---
