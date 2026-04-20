# TTI Helper — Phase 3 Plan & Strategy

Planning artifact captured 2026-04-19. Supersedes the brief Phase 3 outline in `ROADMAP.md`.

**Status:** planning only — no implementation until Phase 2 E2E testing is complete (see bottom).

This document captures decisions made, constraints accepted, and questions still open for Phase 3.
The purpose is to have a single reference when design and implementation start, so we don't re-argue resolved points.

---

## 1. Phase 3 scope (what we're building)

Two new surfaces, plus back-end additions:

| Surface | Users | Notes |
|---|---|---|
| Admin portal | TTI Helper internal staff (~5 users) | Manages clients, devices, firmware. Low traffic, high privilege. |
| Client portal | Customer companies, multi-tenant SaaS | Self-service. Many users, scoped to their own tenant. |
| Back-end additions | n/a | New API Gateway endpoints, DynamoDB tables, second Cognito pool, IoT Jobs for OTA, etc. |

Mobile app and IoT firmware continue to evolve but are not the focus of Phase 3.

---

## 2. Repository strategy

| Repo | Purpose |
|---|---|
| `tti-helper-aws` (existing) | All back-end. Multi-stack CDK: `IotCoreStack` (existing), `AuthStack`, `AdminApiStack`, `ClientApiStack`, `SharedDataStack`. |
| `tti-helper-mobile` (existing) | Flutter app. |
| `tti-helper-iot` (existing) | Firmware. |
| `tti-helper-portal-admin` (new) | Internal back-office console. English only. Private hosting (CloudFront + WAF IP-allowlist). |
| `tti-helper-portal-client` (new) | Multi-tenant customer portal. i18n from day one. Public hosting. |

**Why two portal repos, not a monorepo:** admin and client are *different products* — different users, deploy cadence, security posture, and design investment. Keeping them separate prevents accidental cross-contamination (e.g., tenant-scoping bugs, internal utilities leaking to the client bundle).

**Shared code between portals:** auto-generate a TypeScript API client from an OpenAPI spec in `tti-helper-aws`, publish to a private registry (GitHub Packages or AWS CodeArtifact), consume from both portals. No shared-component monorepo.

---

## 3. Multi-tenancy model — pool

**Decision: pool model** — shared tables with `tenant_id` partitioning.

### Non-negotiable rules

1. **`tenant_id` is a partition key, always.** DynamoDB: `PK = tenant_id#<entity>` or composite `(tenant_id, entity_id)`. No table multi-tenant data without `tenant_id` in the key.
2. **Tenant scoping lives in the data-access layer, not in handlers.** Repository pattern: `TenantScopedRepo(tenantId)` injects `tenant_id` into every read/write. Handlers never touch raw queries.
3. **`tenant_id` is a JWT claim, not a request parameter.** Cognito pre-token-generation Lambda injects `tenant_id` into the ID token from a `Users` table lookup. API Gateway authorizer extracts it. Never trust a `tenant_id` sent in a request body.

### Things to build in from the start

- **IoT topic scoping**: topic scheme becomes `dt/esp32/ttireaderv1/<tenant_id>/<thingID>` so the IoT policy can enforce per-tenant MQTT isolation. Devices get provisioned with a tenant-scoped cert.
- **Per-tenant API Gateway usage plans**: rate + quota limits per tenant to contain noisy neighbors.
- **Tenant-isolation test harness**: a CI test that, for every API endpoint, asserts "as tenant A, cannot see tenant B's rows". Single most effective defense against tenant-leak bugs.
- **Silo escape hatch**: design data access so one whale tenant can later move to an isolated stack via data export + `tenant_id` rewrite, without code changes.

### Admin portal differences

Admin queries explicitly opt out of tenant scoping via an `UnscopedRepo()` class, only instantiated in the admin-API Lambda. Never in client-API code.

---

## 4. Regions & markets — v1

### Launch scope

| Area | Decision |
|---|---|
| AWS region | `us-east-2` only |
| Markets | US + Latin America **excluding Brazil** (Mexico, Argentina, Chile, Colombia, Peru, Ecuador, Venezuela, etc.) |
| Excluded markets | Brazil, Canada, EU, UK, Middle East, Africa, Asia |

### Accepted tradeoff

LATAM users (Argentina, Chile, Peru, etc.) will see ~150–200ms RTT to `us-east-2`. Acceptable for device management and alarm feed in v1. Add `sa-east-1` later when any of the triggers below fires.

### Triggers for adding a second region

- First signed LATAM customer on an enterprise-tier SLA
- First customer complaint about alarm latency with supporting evidence
- Reaching 10+ paying LATAM tenants (arbitrary threshold — revise when approached)
- Any customer in a country with data-residency law that forbids US storage

### Cheap insurance for future regions (build in from day one)

- Tenant record has a `region` attribute, always `us-east-2` in v1. Adding a region later = data migration, not schema change.
- Mobile app reads its AWS config (endpoint, Cognito pool, region) from the tenant payload at login. Hardcoded in v1 but the code path exists.
- CDK stacks accept region as a parameter even when we deploy only one.

---

## 5. Internationalization (i18n)

### Languages for v1

| Language | Coverage | Priority |
|---|---|---|
| English | US, internal staff | P0 |
| Spanish (es-419, LATAM neutral) | Mexico, Chile, Argentina, Colombia, Peru, etc. | P0 |

Admin portal: English only. Internal staff.
Client portal + mobile app: English + Spanish.

**Deferred:** Portuguese (no Brazil), French (no Canada), RTL languages (no Middle East).

### i18n infrastructure (non-negotiable from day one)

Retrofit after the fact is much more expensive than up-front setup.

1. **Externalized strings.** Zero hardcoded user-facing text. Catalogs in `.arb` (Flutter) and JSON (web).
2. **Parameterized messages.** ICU MessageFormat for plurals/gender. Never string-concat sentences.
3. **Locale-aware formatting.** Dates, numbers, currency via locale-aware APIs (`intl` in Flutter, `Intl` on web). UTC in storage, local display.
4. **Locale detection & override.** Device locale at startup, English fallback, user-override persisted.

### Stack choices

- **Mobile (Flutter):** `flutter_localizations` + `intl` + `.arb` files. `flutter gen-l10n` for typed accessors.
- **Client portal (web):** pick one during design — `next-intl` (if Next.js), `react-intl`, or `i18next`. JSON catalogs per locale.
- **Back-end:** error *codes* over the wire, not error messages. Portals translate codes.

---

## 6. Compliance baseline

### In scope for v1

- **US — CCPA / CPRA** (California) plus growing state patchwork (Texas, Virginia, Colorado, etc.)
- **Mexico — LFPDPPP**
- **LATAM general** — Argentina PDPL, Chile Law 19.628 (modernization in flight), Peru, Colombia, etc. All GDPR-adjacent in spirit.

### Practical target

Build a **"LATAM GDPR-lite" baseline** that satisfies ~90% of LATAM + US requirements:

- Consent capture at signup and on material changes
- Data-export endpoint (self-service in client portal)
- Data-erasure endpoint (self-service with hard-delete workflow)
- Audit log of admin access to tenant data
- Privacy policy, cookie notice on client portal
- DPA (Data Processing Agreement) template for enterprise customers
- Data Processing Inventory — sub-processors (Cognito, SES, SNS, S3, DynamoDB) documented

Clean foundation for adding EU (GDPR) or Brazil (LGPD) later.

### Out of scope for v1

- GDPR (no EU users)
- LGPD (no Brazil)
- PIPEDA / Quebec Law 25 / Bill 96 (no Canada)
- Country-specific data-localization laws (Russia, China, Saudi, India)

---

## 7. Back-end shape

### CDK stack decomposition (all in `tti-helper-aws`, all in `us-east-2`)

| Stack | Contents |
|---|---|
| `IotCoreStack` (existing) | IoT Core endpoint, device certs, IoT policies |
| `AuthStack` | Two Cognito User Pools — Admin pool, Client pool (tenant-scoped via pre-token-generation Lambda) |
| `AdminApiStack` | API Gateway + Lambdas for admin portal endpoints; unscoped data access |
| `ClientApiStack` | API Gateway + Lambdas for client portal endpoints; tenant-scoped data access |
| `SharedDataStack` | DynamoDB tables, S3 firmware bucket, shared queues |
| `OtaStack` | IoT Jobs, firmware manifest pipeline |

All stacks accept `region` and `env` (dev/staging/prod) as parameters. v1 deploys `us-east-2` / `prod` only.

### Cognito model

- **Admin pool:** single pool, us-east-2, all users are TTI staff.
- **Client pool:** single pool, us-east-2 (v1). Users scoped by `tenant_id` JWT claim injected by pre-token-generation Lambda.
- If we later add `sa-east-1`, client-pool strategy becomes regional pools with tenant → pool routing. Not a v1 concern.

---

## 8. Decision log (answered)

| # | Question | Decision |
|---|---|---|
| 1 | Portal repo strategy | Two separate repos (admin + client), OpenAPI client for sharing |
| 2 | Back-end repo | Single `tti-helper-aws`, multi-stack CDK |
| 3 | Multi-tenancy model | Pool (`tenant_id` in shared tables) |
| 4 | Launch regions | `us-east-2` only |
| 5 | Launch markets | US + LATAM ex-Brazil |
| 6 | Launch languages | English + Spanish (es-419) |
| 7 | Admin portal i18n | No — English only |
| 8 | Client portal i18n | Yes — infrastructure mandatory from day one |
| 9 | GDPR/LGPD in v1 | Deferred — no EU, no Brazil |
| 10 | Canada in v1 | Out of scope |
| 11 | Quebec-specific work (Law 25, fr-CA) | Out of scope |

---

## 9. Open questions (still to resolve before design)

| # | Question | Why it matters |
|---|---|---|
| A | Tenant-to-region binding mechanics | Even though v1 is single region, decide now whether tenants pick at signup or we auto-assign. |
| B | Firmware region binding | Single binary with runtime region discovery, or per-region firmware? Keeping single-binary is simpler if we later go multi-region. |
| C | OTP channel for LATAM users | SMS via SNS (expensive per message in LATAM) vs email via SES. Probably email-first with SMS opt-in. |
| D | Admin portal hosting | CloudFront + WAF IP-allowlist, or put it behind a VPN? IP-allowlist is simpler. |
| E | OpenAPI toolchain | Speccy/Stoplight/TypeSpec? Decide once. Output: TypeScript client + Python server stubs. |
| F | Billing model | Needed for client-portal signup flow. Out of Phase 3 scope? |
| G | Offline tolerance for mobile in LATAM | Confirm MQTT retry and queue behavior before launch. |
| H | Enterprise contract language | Some LATAM jurisdictions require Spanish for consumer/enterprise contracts. Legal review before first enterprise deal. |

---

## 10. What must happen before we start Phase 3

1. **E2E test Phase 2** (inspection feature) on real hardware and real client scenarios. Current top priority.
2. **SQS message buffer (#22 in `ROADMAP.md`)** — blocking dependency for Phase 3. No point building a portal if Phase 2 is lossy.
3. **Resolve the open questions above** — at minimum A, B, C, E before any CDK or portal code lands.
4. **Decide launch timeline** — which triggers ship client portal vs. admin portal first?

Until those gates are cleared, Phase 3 is planning only.
