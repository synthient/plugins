---
name: synthient-migrate
description: Migrate a codebase off another IP intelligence vendor onto Synthient. Audits every call site, produces a field-by-field mapping report with the semantic changes called out, then rewrites the integration. Use when moving off Spur, IPQualityScore, MaxMind, ipinfo, ipdata, Castle, Fingerprint, or any proxy or VPN detection API.
---

# Migrate to Synthient

Migrating an IP intelligence integration is not a find-and-replace. The field tables map names; they do not map **meaning**. A ported threshold constant, a copied boolean check, or a reused provider-name string will compile, pass review, and silently change who gets blocked.

So: audit, map, **report before touching anything**, rewrite, verify.

If the user did not name a vendor, detect it from the codebase and confirm before proceeding.

## Phase 1: Audit

Sweep the repository. Use a subagent if the codebase is large enough that the search would crowd out the rest of the work. Do not stop at the first hit; integrations sprawl.

**Dependencies.** Vendor SDK packages in `package.json`, `go.mod`, `requirements.txt`, `pyproject.toml`, `Gemfile`, `pom.xml`, `Cargo.toml`, and lockfiles.

**Network identifiers.** Base URLs and hostnames anywhere: source, config, infrastructure-as-code, CI, allowlists, egress rules. Known ones: `api.spur.us`, `ipqualityscore.com`, `geoip.maxmind.com`, `ipinfo.io`, `api.ipdata.co`.

**Credentials.** Env var names, secret manager keys, `.env` files and examples, CI secrets, Terraform variables. Known ones: `SPUR_TOKEN`, `IPQS_KEY`, `MAXMIND_LICENSE_KEY`.

**Call sites.** Every invocation, SDK or raw HTTP. Look for wrappers; most codebases have an internal client, and the real surface is its callers.

**Fields actually consumed.** Not what the vendor returns, what this codebase *reads*. This is the real migration surface and it is usually a fraction of the response.

**Decision logic.** Every threshold, boolean check, string comparison against a provider name, and switch over vendor enums. This is where semantic changes do their damage. Quote the constants verbatim.

**Persisted values.** Database columns, caches, and analytics dimensions holding vendor-specific strings or scores. These outlive the code change and are the most-missed item in a migration.

**Tests.** Fixtures, mocks, cassettes, stubs.

## Phase 2: Map

For **Spur** or **IPQualityScore**, fetch the authoritative field table:

- `https://docs.synthient.com/migration/spur.md`
- `https://docs.synthient.com/migration/ipqs.md`

Then read the matching file in `references/` for the semantic deltas the tables cannot express: `references/spur.md` or `references/ipqs.md`. These are the parts that break silently.

For **any other vendor**, follow `references/generic.md`. It derives a mapping from the audited fields, using `https://docs.synthient.com/ipapi.md` as the target shape.

Load the Synthient response shape and field vocabularies from the `synthient` skill rather than restating them.

## Phase 3: Report

Present before editing. The user decides what ships.

- **Direct mappings**: same meaning, different name. Safe.
- **Semantic changes**: field maps, meaning shifts. Each one names the decision logic it affects and what needs re-tuning. This is the section that matters.
- **No equivalent**: vendor fields with nothing on the Synthient side, and what to do instead.
- **New signals**: what Synthient adds, and whether it is worth adopting now or later.
- **Plumbing changes**: auth header, env var, endpoint, batching, rate limits, credit model.

Mark every row with confidence. Anything inferred rather than read from the migration guide gets flagged as inferred.

## Phase 4: Rewrite

Plumbing first, so the rest has something to run against.

1. **Auth and transport.** Header to `x-api-key`, endpoint to `https://api.synthient.com/api/v4`, env var to `SYNTHIENT_API_KEY`. In Go use `github.com/synthient/go-synthient/v2`. In every other language write plain HTTP; Python, Node, Java, and Ruby SDKs do not exist yet, so do not import one.
2. **Response types.** Model the `network` / `location` / `intelligence` structure. Do not flatten it to match the old vendor's shape; that moves the migration into the type layer where it is harder to see.
3. **Decision logic.** Apply the semantic changes from the report. Where a threshold needs re-tuning, leave the old value with a clear marker rather than inventing a new one, and say it needs calibration.
4. **Batching.** If the old integration looped over addresses, switch to `POST /api/v4/lookup/ips`, up to 1,000 per request at a 10% discount.
5. **Benign automation.** Add the guard if it was not there: never block on `SEARCH_ENGINE`, `AI_CRAWLER`, `SOCIAL_MEDIA`, `UPTIME_MONITOR`, `LINK_PREVIEW`, `SEO_CRAWLER`, `WEB_ARCHIVER`, `WEBHOOK_PROVIDER`, or `PAYMENT_PROCESSOR`. Most vendor migrations introduce this bug because the old vendor lumped crawlers in with abuse.

Offer to keep the old client behind a feature flag, or to run both in shadow mode and log disagreements. For anything with a tuned threshold, shadow mode is the honest recommendation; it is the only way to recalibrate against real traffic rather than guessing.

## Phase 5: Verify

- Update test fixtures to real Synthient response shapes. A fixture still shaped like the old vendor will pass while production breaks.
- Run the test suite.
- Run the `synthient-doctor` skill to confirm credentials, scopes, and remaining credits.
- Do a live lookup against a known address and check the parsed result end to end.
- Grep for leftovers: the old base URL, env var, SDK import, and field names in comments and docs.
