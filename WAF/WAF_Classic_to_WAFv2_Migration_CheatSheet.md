# AWS WAF - Classic to WAFv2 Migration Cheat Sheet

## Overview

**AWS WAF Classic** is the original (2015-era) API. **AWS WAF (WAFv2)** launched in 2019 with a redesigned API, more capabilities, and lower cost model. AWS recommends migrating off Classic; new features (Bot Control, ATP/ACFP, labels, JSON body inspection, rate custom keys) are **WAFv2 only**.

**Key point:** WAFv2 is a different service namespace (`wafv2`) with different resources. Migration is not just a rename — conditions become statements, and some Classic concepts don't map 1:1.

---

## Classic vs WAFv2 Terminology

| WAF Classic                     | WAFv2 equivalent                              |
| ------------------------------- | --------------------------------------------- |
| **Conditions** (e.g., SqlInjectionMatchSet) | **Rule statements** (SqliMatchStatement) |
| **Rules** (list of conditions)  | **Rules** (statements + action)               |
| **WebACL**                      | **Web ACL** (new resource)                    |
| **Rule Groups** (limited)       | **Rule groups** (managed + custom, richer)    |
| **AWS/Marketplace rules**       | **Managed rule groups**                       |
| Separate global vs regional APIs (`waf` / `waf-regional`) | Single API with `--scope CLOUDFRONT` / `REGIONAL` |
| IP match / byte match / SQLi / XSS / size / geo | Same, expressed as statements           |
| No labels                       | **Labels** (rule-to-rule signaling)           |
| No JSON body inspection         | **JsonBody** field                            |
| Basic rate rules                | Rate-based with **custom keys / windows**     |
| No CAPTCHA/Challenge            | **CAPTCHA & Challenge** actions               |

---

## Why Migrate

| Benefit                        | Detail                                                        |
| ------------------------------ | ------------------------------------------------------------- |
| **More features**              | Bot Control, ATP/ACFP, labels, JSON inspection, CAPTCHA       |
| **Better managed rules**       | Actively maintained AWS managed rule groups                   |
| **Simpler pricing**            | WCU-based, generally cheaper                                  |
| **Single API + IaC friendly**  | One namespace, better CloudFormation/Terraform support        |
| **Ongoing investment**         | Classic receives no new features                              |

---

## Migration Approaches

| Approach                        | When to use                                          |
| ------------------------------- | ---------------------------------------------------- |
| **Automated migration script/wizard** | Straightforward Classic Web ACLs               |
| **Rebuild from scratch (recommended)** | Take the chance to adopt managed rule groups   |
| **Hybrid / phased**             | Large environments; migrate resource-by-resource     |

> AWS provides a migration API/tooling that generates a CloudFormation template from a Classic Web ACL. Review the output — some conditions need manual attention.

---

## Automated Migration (High Level)

```
1. Inventory Classic Web ACLs, rules, conditions
        |
        v
2. Run migration tooling → generates a WAFv2 CloudFormation template
        |
        v
3. Review & adjust template (managed rules, labels, unsupported items)
        |
        v
4. Deploy the WAFv2 Web ACL (in parallel, Count mode)
        |
        v
5. Enable logging; compare WAFv2 vs Classic behavior on live traffic
        |
        v
6. Switch resource association from Classic → WAFv2
        |
        v
7. Decommission the Classic Web ACL
```

---

## Step-by-Step Checklist

1. **Inventory** every Classic Web ACL and the resources it protects (CloudFront, ALB, API GW).
2. **Map conditions to statements** — SQLi/XSS/size/geo/IP/byte conditions become their WAFv2 statement equivalents.
3. **Replace custom signatures with managed rule groups** where possible (less to maintain).
4. **Rebuild rate rules** using WAFv2 rate-based rules (consider custom keys / window).
5. **Build the WAFv2 Web ACL in parallel** — do not detach Classic yet.
6. **Deploy WAFv2 in Count mode** and enable full logging.
7. **Compare** WAFv2 sampled/log data against Classic for false positives/negatives.
8. **Flip WAFv2 rules to Block**, then **associate the resource** with the WAFv2 Web ACL.
9. **Monitor** closely for a period.
10. **Disassociate and delete** the Classic Web ACL once confident.

---

## Association Cutover Notes

| Resource     | Cutover action                                                        |
| ------------ | --------------------------------------------------------------------- |
| **CloudFront** | Update distribution `WebACLId` to the WAFv2 Web ACL ARN             |
| **ALB / API GW / AppSync** | `wafv2 associate-web-acl` (replaces Classic association) |

> A resource can only be tied to one Web ACL (Classic **or** WAFv2). Associating WAFv2 effectively replaces Classic on that resource.

---

## CLI: Distinguishing the APIs

```bash
# Classic (global / CloudFront)
aws waf list-web-acls

# Classic (regional)
aws waf-regional list-web-acls --region us-east-1

# WAFv2 (regional)
aws wafv2 list-web-acls --scope REGIONAL --region us-east-1

# WAFv2 (CloudFront — always us-east-1)
aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1
```

> If commands use `aws waf` or `aws waf-regional`, you're on **Classic**. `aws wafv2` is the current service.

---

## Things That Don't Map 1:1

| Classic item                    | Migration note                                                   |
| ------------------------------- | ---------------------------------------------------------------- |
| Rule priority semantics         | Re-verify order in WAFv2 (evaluation is priority-based)          |
| Marketplace rules               | Re-subscribe to the WAFv2 versions of vendor rule groups         |
| Custom regex handling           | Move to WAFv2 regex pattern sets; re-validate patterns           |
| Size constraints                | Re-express as SizeConstraintStatement + OversizeHandling         |
| Rate limits                     | Rebuild; WAFv2 windows/keys differ                               |
| Logging setup                   | Reconfigure WAFv2 logging (new `aws-waf-logs-` destinations)     |

---

## Gotchas & Caveats

1. **Classic and WAFv2 are different APIs** — `aws waf` / `aws waf-regional` (Classic) vs `aws wafv2` (current). Tooling, ARNs, and resources do not interchange.
2. **A resource can only have one Web ACL** — associating a WAFv2 Web ACL replaces the Classic one on that resource; there's no running both simultaneously on the same resource.
3. **The auto-migration template is a starting point, not a drop-in** — review it; some conditions don't map cleanly and need manual work.
4. **Managed rule groups are stricter than typical Classic setups** — expect new false positives (e.g., `SizeRestrictions_BODY`); tune in Count before enforcing.
5. **Marketplace rules must be re-subscribed** in their WAFv2 form — Classic subscriptions don't carry over.
6. **Rate rules must be rebuilt** — WAFv2 windows/keys differ; don't assume the same numeric limit behaves identically.
7. **Logging must be reconfigured** — Classic logging config doesn't apply; set up WAFv2 logging with `aws-waf-logs-` destinations.
8. **Priority/evaluation semantics can shift** — re-verify rule order after migration; a misordered Allow can neutralize blocks.
9. **Keep Classic until WAFv2 is proven** — but remember you may be billed for both during overlap; decommission promptly after cutover.
10. **CloudFront cutover is via distribution config**, not `associate-web-acl` — update `WebACLId` to the WAFv2 ARN.
11. **WAFv2-only features have no Classic equivalent** — labels, JSON body inspection, CAPTCHA/Challenge, Bot Control/ATP/ACFP, custom rate keys. Don't expect to "port" them; you're adding them.
12. **Don't leave orphaned Classic resources** — dangling Classic Web ACLs/rules keep accruing charges and cause confusion.

### Official Caveats (from AWS docs)

**Migration tool caveats & limitations:**
1. **Single-account only.** You can migrate Classic resources only to WAFv2 resources **in the same account**.
2. **Web ACL configurations only.** The migration brings over Web ACLs and the resources they use. A rule group or IP set **not used by a migrated Web ACL must be recreated manually** in WAFv2.
3. **No AWS Marketplace managed rules.** These aren't migrated — re-subscribe to equivalent WAFv2 versions (and review the free AWS Managed Rules first).
4. **No resource associations.** By design, the migrated Web ACL is **not** associated with any protected resources — associate it yourself after verifying the migration.
5. **Logging is disabled on the migrated Web ACL** by design — enable it when you're ready to cut over.
6. **No Firewall Manager rule groups.** You can migrate an FMS-managed Web ACL, but the rule group isn't brought over — **recreate the FMS policy** for the new WAF instead of using the migration tool for those.
7. **Don't migrate AWS WAF Security Automations.** The migration doesn't convert the Lambda functions those automations use — deploy the automations for the latest version instead.

**Version / lifecycle facts:**
8. **AWS WAF Classic support ended September 30, 2025**, and Classic is going through a planned end-of-life (check your AWS Health dashboard for Region-specific dates). Migrate off Classic.
9. **Separate APIs / namespaces.** WAFv2 (`wafv2`) can't access resources created in Classic; Classic resources are reachable only through the Classic APIs (`waf` / `waf-regional`), which kept their prior names and endpoints.

> Sources: [Migration caveats and limitations](https://docs.aws.amazon.com/waf/latest/developerguide/waf-migrating-caveats.html) and [Why migrate to AWS WAF?](https://docs.aws.amazon.com/waf/latest/developerguide/waf-migrating-why-migrate.html). Content was rephrased for compliance with licensing restrictions.

---

## Troubleshooting

| Issue                                        | Cause                                        | Fix                                                       |
| -------------------------------------------- | -------------------------------------------- | --------------------------------------------------------- |
| Migrated Web ACL behaves differently         | Condition→statement gaps / order changes     | Compare in Count mode with logging before cutover         |
| Resource still protected by Classic          | Association not switched                     | Associate WAFv2 Web ACL (replaces Classic)                |
| Marketplace rule missing after migration     | Not re-subscribed in WAFv2                    | Subscribe to the WAFv2 vendor rule group                  |
| CloudFormation template errors               | Auto-generated template needs edits          | Review/adjust unsupported conditions manually             |
| Increased false positives post-migration     | Managed rule groups stricter than old rules  | Override noisy rules to Count; tune with logs             |
| Logging stopped working                      | Old Classic logging config                   | Set up WAFv2 logging with `aws-waf-logs-` destinations    |

---

## Best Practices

1. **Run WAFv2 in parallel with Classic** in Count mode before cutover.
2. **Enable logging on WAFv2 first** and compare against Classic behavior.
3. **Adopt managed rule groups** instead of porting custom signatures where possible.
4. **Migrate one resource at a time** in large environments.
5. **Keep Classic until WAFv2 is proven**, then decommission to avoid double billing.
6. **Rebuild rate rules deliberately** — take advantage of custom keys/windows.
7. **Re-subscribe to Marketplace rules** in their WAFv2 form.
8. **Capture everything as IaC** during migration for repeatability and rollback.

---

## Useful Links

- [Migrating from WAF Classic to WAFv2](https://docs.aws.amazon.com/waf/latest/developerguide/waf-migrating-from-classic.html)
- [Migration Procedure & Tooling](https://docs.aws.amazon.com/waf/latest/developerguide/waf-migrating-procedure.html)
- [What Changed in WAFv2](https://docs.aws.amazon.com/waf/latest/developerguide/classic-waf-chapter.html)
- [WAF Classic Documentation](https://docs.aws.amazon.com/waf/latest/developerguide/classic-waf-chapter.html)
- [Migrate Your AWS WAF Classic (Blog)](https://aws.amazon.com/blogs/security/migrate-your-web-application-firewall-rules-to-aws-waf/)

---
