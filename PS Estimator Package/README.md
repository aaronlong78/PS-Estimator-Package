# PS Estimator

A Lightning Web Component (`psQuoteBuilder`) on the **Quote** record page that builds a
**professional-services staffing estimate**: roles, hours by week and phase, bill rates, costs, and
margin. On save it writes one **$0/mo anchor quote line per role** with negotiated hourly usage rates
to the Quote, using Revenue Cloud (RLM) usage-based pricing, and stores the estimated cost, revenue,
and margin on the Quote.

> **Target org:** a Revenue Cloud / QuantumBit demo, trial, or Developer Edition org that is
> identical to the source org. **Never deploy to production** from this package.

For the design and data model, see [docs/ps-ramp-architecture.md](docs/ps-ramp-architecture.md).

---

## What's included

```
force-app/main/default/
  classes/             # PsQuoteController, PsQuoteService, PsCostBook, PsQuoteControllerTest
  lwc/                 # psQuoteBuilder (PS Estimator), psRoleLookup
  objects/             # PS_Estimate_Segment__c, PS_Quote_Settings__mdt, Quote + QuoteLineItem fields
  customMetadata/      # PS_Quote_Settings.Default
  flexipages/          # PS_Quote_Record_Page (contains psQuoteBuilder)
  pages/               # PsSessionId (Visualforce, session for Connect API callouts)
  permissionsets/      # PS_Quote_Builder
  remoteSiteSettings/  # PsRevCloudSelf (same-org callout; set per target org)
manifest/package.xml   # every deployable component, listed explicitly
SKILL.md               # demo data seeding skill (run by an AI agent after deploy)
sf-objects-reference.md
scripts/               # set-quote-app-default.sh, update-theme-brand-image.sh
docs/                  # architecture notes
AGENTS.md, CLAUDE.md   # agent instructions (use the local skill, deploy steps)
```

Deploy mechanism: `sf project deploy start --manifest manifest/package.xml`.

## Prerequisites

| Requirement | Check |
|---|---|
| Salesforce CLI (`sf`) 2.x | `sf --version` |
| Target org authenticated + aliased | `sf org display --target-org <alias>` |
| `python3` (used by the scripts) | `python3 --version` |
| Revenue Cloud enabled, with an active **Term Monthly** selling model, a **Base** rate card, and an **Arrears** billing policy | Setup / Product Catalog |
| The `RLM_*` Quote fields and actions used by the record page (standard in QuantumBit / RLM template orgs) | Object Manager → Quote |
| Connect API version **v68.0** supported (`PsQuoteService.API_VERSION`) | `sf org display` → `apiVersion` |

## Deploy

```bash
git clone <https://github.com/aaronlong78/PS-Estimator-Package.git> "PS Estimator Package"
cd "PS Estimator Package"

# 1. Point the Remote Site Setting at the target org
sf org display --target-org <alias> --json    # copy result.instanceUrl
#    Set <url> in force-app/main/default/remoteSiteSettings/PsRevCloudSelf.remoteSite-meta.xml
#    to that instanceUrl (https://<mydomain>.my.salesforce.com)

# 2. Validate, then deploy
sf project deploy start --manifest manifest/package.xml --target-org <alias> --dry-run --wait 30
sf project deploy start --manifest manifest/package.xml --target-org <alias> --wait 30

# 3. Run the tests explicitly (trial / Developer Edition orgs may skip them during deploy)
sf apex run test --class-names PsQuoteControllerTest --code-coverage --result-format human --wait 20 --target-org <alias>
```

## Post-deploy steps

1. **Assign the permission set** (custom fields have no access until this is done):
   `sf org assign permset --name PS_Quote_Builder --target-org <alias>`
2. **Make PS Quote Record Page the app default** for the Revenue Cloud app:
   `bash scripts/set-quote-app-default.sh <alias>`
   The script edits only the Quote override on the target org's own app and is safe to re-run.
   Manual alternative: Lightning App Builder → open **PS Quote Record Page** → Activation →
   App Default → Revenue Cloud → Save.
3. **Optional:** set `PS_Quote_Settings.Default` (Setup → Custom Metadata Types).
   `Cost_Book_Name__c` picks a cost book by name (blank = the org's default cost book), and
   `Target_Margin_Pct__c` sets the target margin.

## Seed demo data

The builder needs staffable roles with rates and costs. Open this folder in **Cursor** or
**Claude Code** and ask the agent to *"seed PS quoting demo data"*. It follows the local
[SKILL.md](SKILL.md), which:

- researches a target company and proposes 10 PS roles with hourly bill rates and costs in a canvas for review
- replaces the org's theme brand image with the company logo
- creates the role products, $0 Term Monthly price book entries, the shared `PS_LABOR_HOURS` usage
  resource, Negotiable Base rate card entries, and a default cost book with hourly costs
- assigns the permission set, checks the Remote Site URL, and sets the Quote app default

`AGENTS.md` / `CLAUDE.md` tell the agent to use this local copy of the skill, not a globally installed
one. Field-level details are in [sf-objects-reference.md](sf-objects-reference.md).

## Verify

```bash
sf project deploy report --target-org <alias>
```

Check that the roles are visible to the builder (expect one row per seeded role):

```sql
SELECT ProductId, Rate, RateNegotiation FROM RateCardEntry
WHERE UsageResource.Code = 'PS_LABOR_HOURS' AND RateCard.Type = 'Base' AND ProductId != null
```

Then open a Quote in the **Revenue Cloud** app → add roles and hours in PS Estimator → save →
confirm the quote lines appear and **PS Estimated Cost / Revenue / Margin / Margin %** are filled in
on the Quote.

## Known gotchas & limitations

- **Remote Site URL is per org.** `PsRevCloudSelf` holds whichever org was deployed last. If it
  doesn't match the target org's `instanceUrl`, the save/build callouts fail.
- **The deploy is atomic.** If `PS_Quote_Record_Page` is re-retrieved or edited in a dev org, it can
  pick up header actions or Work Guide properties that a fresh org rejects (e.g.
  `Quote.RLM_CreateContract`, `isAutoAdvanceToNextWorkItem`), and one bad reference rolls back
  everything. Always run the `--dry-run` first.
- **No field access without the permission set,** including for System Administrator.
- **Never add the Revenue Cloud `CustomApplication` to the manifest.** Deploying the whole app
  overwrites that org's navigation and page assignments; use `scripts/set-quote-app-default.sh`.
- **Apex tests may be skipped** on trial and Developer Edition orgs; run `PsQuoteControllerTest` yourself.

## Safety

- The deploy is metadata only. It loads and deletes no data.
- The seeding skill **creates** products, price book entries, usage resources, rate card entries,
  and cost book entries, and **replaces** the theme brand image. Run it on demo orgs only.
- Saving an estimate in PS Estimator writes quote lines and estimate rows to that Quote. Try it on a
  test quote first.

## Screenshots

<!-- Add PS Estimator screenshots here. -->
