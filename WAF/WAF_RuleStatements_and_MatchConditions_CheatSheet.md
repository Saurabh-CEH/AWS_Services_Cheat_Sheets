# AWS WAF - Rule Statements & Match Conditions Cheat Sheet

## Overview

A **rule statement** is the heart of a WAF rule — it defines *what* to inspect in a request and *how* to match. Statements can be simple (one condition) or combined with logical operators (AND / OR / NOT) and nested.

**Key point:** Statements are composable. You inspect a **request component** using a **match type**, optionally after applying **text transformations**.

---

## Statement Categories

| Category            | Statements                                                                 |
| ------------------- | -------------------------------------------------------------------------- |
| **Match statements**| ByteMatch, RegexPatternSet, RegexMatch, SizeConstraint, SqliMatch, XssMatch |
| **Location match**  | IPSetReference, GeoMatch                                                    |
| **Rate-based**      | RateBasedStatement (see Rate-Based Rules sheet)                            |
| **Reference**       | ManagedRuleGroupStatement, RuleGroupReferenceStatement, LabelMatchStatement |
| **Logical**         | AndStatement, OrStatement, NotStatement                                    |

---

## Request Components You Can Inspect

| Component                  | Description                                              |
| -------------------------- | -------------------------------------------------------- |
| **SingleHeader**           | One named header (e.g., `User-Agent`, `Cookie`)          |
| **Headers**                | All headers (with match scope: keys/values/all)          |
| **Cookies**                | Cookie header parsed into name/value pairs               |
| **SingleQueryArgument**    | One named query-string argument                          |
| **AllQueryArguments**      | All query-string arguments                               |
| **QueryString**            | Raw query string                                         |
| **UriPath**                | The path portion of the URI                              |
| **Method**                 | HTTP method (GET, POST, ...)                             |
| **Body**                   | Request body (subject to size limits)                    |
| **JsonBody**               | Parsed JSON body (target specific keys/values)           |
| **JA3Fingerprint**         | TLS client fingerprint                                   |
| **HeaderOrder**            | Order of headers (bot signal)                            |

---

## Match Types (ByteMatch)

For string / byte matches you pick a **positional constraint**:

| Constraint            | Meaning                                    |
| --------------------- | ------------------------------------------ |
| **EXACTLY**           | Component equals the search string         |
| **STARTS_WITH**       | Component begins with the string           |
| **ENDS_WITH**         | Component ends with the string             |
| **CONTAINS**          | String appears anywhere                    |
| **CONTAINS_WORD**     | String appears as a whole word             |

---

## Text Transformations

Applied **before** matching to normalize evasion attempts. You can chain several (they run in priority order).

| Transformation        | Effect                                                  |
| --------------------- | ------------------------------------------------------- |
| **NONE**              | No change                                               |
| **LOWERCASE**         | Convert to lowercase                                    |
| **URL_DECODE**        | Decode URL-encoded characters                           |
| **HTML_ENTITY_DECODE**| Decode HTML entities (`&lt;` → `<`)                     |
| **COMPRESS_WHITE_SPACE** | Collapse whitespace                                  |
| **REMOVE_NULLS**      | Strip null bytes                                        |
| **CMD_LINE**          | Normalize command-line tricks                           |
| **BASE64_DECODE**     | Decode Base64                                           |
| **URL_DECODE_UNI**    | Decode Unicode URL encoding                             |
| **REPLACE_COMMENTS**  | Remove comments (SQL/HTML)                              |

> Always add transformations that match how an attacker could obfuscate (e.g., URL_DECODE + LOWERCASE for SQLi/XSS).

---

## Logical Combinations

```
AndStatement:  match only if ALL nested statements match
OrStatement:   match if ANY nested statement matches
NotStatement:  match if the nested statement does NOT match
```

Example logic (allow only US traffic to /admin):

```
Block if:
  AndStatement
    ├── ByteMatch: UriPath STARTS_WITH "/admin"
    └── NotStatement
          └── GeoMatch: country in [US]
```

---

## Common Statement Examples (JSON)

### IP set match

```json
{
  "IPSetReferenceStatement": {
    "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/ipset/blocked-ips/abc"
  }
}
```

### Geo match (block specific countries)

```json
{
  "GeoMatchStatement": {
    "CountryCodes": ["CN", "RU", "KP"]
  }
}
```

### Byte match on URI path

```json
{
  "ByteMatchStatement": {
    "SearchString": "/wp-login.php",
    "FieldToMatch": { "UriPath": {} },
    "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ],
    "PositionalConstraint": "STARTS_WITH"
  }
}
```

### SQL injection match on body

```json
{
  "SqliMatchStatement": {
    "FieldToMatch": { "Body": { "OversizeHandling": "CONTINUE" } },
    "TextTransformations": [
      { "Priority": 0, "Type": "URL_DECODE" },
      { "Priority": 1, "Type": "HTML_ENTITY_DECODE" }
    ],
    "SensitivityLevel": "HIGH"
  }
}
```

### XSS match on a query argument

```json
{
  "XssMatchStatement": {
    "FieldToMatch": { "SingleQueryArgument": { "Name": "q" } },
    "TextTransformations": [
      { "Priority": 0, "Type": "URL_DECODE" },
      { "Priority": 1, "Type": "HTML_ENTITY_DECODE" }
    ]
  }
}
```

### Size constraint (block huge bodies)

```json
{
  "SizeConstraintStatement": {
    "FieldToMatch": { "Body": { "OversizeHandling": "CONTINUE" } },
    "ComparisonOperator": "GT",
    "Size": 8192,
    "TextTransformations": [ { "Priority": 0, "Type": "NONE" } ]
  }
}
```

### Regex pattern set

```json
{
  "RegexPatternSetReferenceStatement": {
    "ARN": "arn:aws:wafv2:us-east-1:123456789012:regional/regexpatternset/bad-agents/abc",
    "FieldToMatch": { "SingleHeader": { "Name": "user-agent" } },
    "TextTransformations": [ { "Priority": 0, "Type": "LOWERCASE" } ]
  }
}
```

### JSON body targeting a key

```json
{
  "SqliMatchStatement": {
    "FieldToMatch": {
      "JsonBody": {
        "MatchPattern": { "IncludedPaths": ["/user/email"] },
        "MatchScope": "VALUE",
        "InvalidFallbackBehavior": "MATCH"
      }
    },
    "TextTransformations": [ { "Priority": 0, "Type": "NONE" } ]
  }
}
```

### Label match (react to a managed rule group label)

```json
{
  "LabelMatchStatement": {
    "Scope": "LABEL",
    "Key": "awswaf:managed:aws:bot-control:bot:category:search_engine"
  }
}
```

---

## SQLi / XSS Sensitivity

| Statement | Field                | Notes                                                       |
| --------- | -------------------- | ----------------------------------------------------------- |
| SQLi      | `SensitivityLevel`   | `LOW` (fewer false positives) or `HIGH` (stricter detection)|
| XSS       | (no sensitivity knob)| Rely on transformations + field selection                   |

---

## Oversize & Invalid Handling

| Option                    | Values                    | Applies to           |
| ------------------------- | ------------------------- | -------------------- |
| **OversizeHandling**      | CONTINUE / MATCH / NO_MATCH | Body, JsonBody, Headers, Cookies |
| **InvalidFallbackBehavior** | MATCH / NO_MATCH / EVALUATE_AS_STRING | JsonBody |

- **CONTINUE** — inspect the portion within the limit, ignore the rest.
- **MATCH** — treat oversize as a match (strict).
- **NO_MATCH** — treat oversize as no match (permissive).

---

## CLI: Testing a Statement Quickly

Statements live inside rules. To iterate, edit a `rules.json` and update the Web ACL:

```bash
aws wafv2 update-web-acl \
  --name my-web-acl --scope REGIONAL \
  --id 12345678-1234-1234-1234-123456789012 \
  --lock-token "$(aws wafv2 get-web-acl --name my-web-acl --scope REGIONAL \
      --id 12345678-1234-1234-1234-123456789012 --region us-east-1 \
      --query LockToken --output text)" \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myWebAcl \
  --rules file://rules.json \
  --region us-east-1
```

---

## Gotchas & Caveats

1. **No transformations = trivially bypassed** — SQLi/XSS matches on raw fields miss URL-encoded, HTML-entity-encoded, or mixed-case payloads. Always chain URL_DECODE + HTML_ENTITY_DECODE + LOWERCASE.
2. **A bare domain/string match is not a regex** — ByteMatch is literal; use a regex pattern set or `RegexMatch` for pattern logic.
3. **Body inspection is size-capped** (ALB/AppSync 8 KB; CloudFront/API GW/Cognito/App Runner/Verified Access/Bedrock AgentCore 16 KB default, up to 64 KB) — content past the limit is only seen if you raise the limit and set `OversizeHandling`.
4. **OversizeHandling default can hide attacks** — with NO_MATCH, oversized bodies are treated as clean. For strict APIs use MATCH.
5. **JSON body matching needs the right MatchScope** — matching KEY vs VALUE vs ALL changes results; a wrong scope silently misses the payload.
6. **`InvalidFallbackBehavior` matters for malformed JSON** — bots send broken JSON on purpose; EVALUATE_AS_STRING or MATCH avoids blind spots.
7. **SQLi `SensitivityLevel: HIGH` is noisy** — it raises false positives on legitimate content (SQL-like text, code snippets). Start LOW.
8. **Transformations run in priority order** — order can change the result (e.g., decode before lowercasing). Set priorities deliberately.
9. **`Headers`/`Cookies`/`AllQueryArguments` inspect many values** and cost more WCU than a single named field — target `SingleHeader`/`SingleQueryArgument` when possible.
10. **Geo match uses WAF's IP geolocation**, which can be wrong or `-` for some IPs — and behind a proxy it geolocates the proxy unless forwarded-IP is configured.
11. **Regex engine is restricted** — some PCRE features (certain lookbehind/backreferences) are unsupported and rejected at create time.
12. **`CONTAINS_WORD` has specific word-boundary semantics** — it is not the same as CONTAINS; test before relying on it.

### Official Caveats (from AWS docs)

1. **Missing component → treated as no match.** Unless otherwise noted, if a request lacks the component a rule inspects, WAF evaluates it as **not matching**.
2. **One component per statement.** Each match statement inspects a single request component; to inspect multiple components, write multiple statements.
3. **Body inspection limits differ by resource.** ALB and AppSync: first **8 KB**. CloudFront, API Gateway, Cognito, App Runner, Verified Access, and Bedrock AgentCore: **16 KB by default, raisable up to 64 KB**. You **must** set oversize handling for Body/JsonBody.
4. **Headers/Cookies are doubly capped.** WAF inspects at most the first **8 KB** *and* the first **200 headers** (or 200 cookies) — whichever limit is hit first. Set oversize handling accordingly.
5. **JSON body parsing doubles WCU**, and **`All query parameters` adds 10 WCU** over the base cost.
6. **JSON parsing does not fully validate JSON.** Parsing can succeed on invalid JSON (missing commas/colons, duplicate keys), and extraction/evaluation results can then be unexpected. Validate JSON in your app; use `Body parsing fallback behavior` (None/Evaluate as string/Match/No match) deliberately.
7. **`Match scope: All` is an OR across keys and values** — it matches if keys OR values match, not both. To require both, combine two statements with a logical AND.
8. **Fallback behavior is required for some components.** URI fragment, JA3, and JA4 require a `fallback behavior`; JA3/JA4 also require logging to be enabled and only work inside an exact string-match statement.
9. **URI fragment / JA3 / JA4 are CloudFront + ALB only.**
10. **Single query parameter name max is 30 chars and is case-insensitive** — `UserName` also matches `username`.
11. **Without a text transformation, components are inspected exactly as received** (no normalization) — e.g., URI path/fragment.

> Source: [Request components in AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-fields-list.html) and [Oversize web request components](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-oversize-handling.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                | Cause                                             | Fix                                                          |
| ------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------ |
| SQLi/XSS rule bypassed               | Missing transformations for the encoding used     | Add URL_DECODE, HTML_ENTITY_DECODE, LOWERCASE                |
| Body match never fires               | Body larger than inspection limit                 | Set OversizeHandling=CONTINUE, raise body limit if needed    |
| Too many false positives (SQLi)      | Sensitivity HIGH on noisy field                   | Lower to LOW, or narrow field selection                      |
| JSON rule misses nested value        | Wrong MatchScope or path                           | Use IncludedPaths + MatchScope=VALUE                         |
| Header match case issues             | No LOWERCASE transformation                        | Add LOWERCASE transformation                                 |
| Statement too costly (WCU)           | Broad regex / many transformations                | Simplify regex, reduce chained transforms                    |

---

## Best Practices

1. **Always normalize** — chain URL_DECODE + HTML_ENTITY_DECODE + LOWERCASE for injection checks.
2. **Target specific fields** — inspecting one query arg is cheaper and more precise than the whole body.
3. **Use JsonBody targeting** for APIs rather than raw Body scanning.
4. **Combine with NOT for allow-exceptions** — e.g., block a rule group except for a trusted path.
5. **Start with LOW SQLi sensitivity**, raise only if attacks slip through.
6. **Mind OversizeHandling** — choose MATCH for strict APIs, CONTINUE for general web.
7. **Keep regex simple** to control WCU cost and avoid catastrophic backtracking.

---

## Useful Links

- [Rule Statements Reference](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statements-list.html)
- [Fields to Match](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-fields.html)
- [Text Transformations](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-transformation.html)
- [SQLi Match](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-sqli-match.html)
- [XSS Match](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-xss-match.html)
- [Geo Match](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-geo-match.html)
- [JSON Body Inspection](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-fields-list.html#waf-rule-statement-request-component-json-body)
- [Logical Rule Statements](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-logical.html)

---
