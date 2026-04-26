# TTI Helper — Phases 3–6 Plan & Strategy

Planning artifact captured 2026-04-19. Supersedes the brief Phase 3 outline in `ROADMAP.md`.

**Status:** planning only — no implementation until Phase 2 E2E testing is complete (see §11).

This document captures decisions made, constraints accepted, and questions still open for the Phase 3–6 program. The purpose is to have a single reference when design and implementation start, so we don't re-argue resolved points.

> **Note on phasing.** This split (3 = aws, 4 = admin, 5 = client, 6 = web) is the current proposed approach and may change. The cross-cutting decisions in §3 apply regardless of how phases are sequenced.

---

## 1. Phase overview

The work is split into four phases by repo, executed roughly in order:

| Phase | Repo | Focus | Why this order |
|---|---|---|---|
| **Phase 3** | `tti-helper-aws` | Back-end foundation: new CDK stacks, second Cognito pool, DynamoDB tables, IoT Jobs OTA, OpenAPI spec | Everything else depends on it |
| **Phase 4** | `tti-helper-portal-admin` | Internal back-office console for TTI staff (~5 users) | TTI staff must be able to provision tenants before clients can sign in |
| **Phase 5** | `tti-helper-portal-client` + `tti-helper-mobile` (new `monitor` build flavor) | TM module (web) + Monitoring product delivered as a second Flutter build flavor of mobile (iOS / Android / Flutter web) | Depends on Phase 3 endpoints + Phase 4 tenant provisioning |
| **Phase 6** | `tti-helper-web` | Public marketing site updates: sign-in/sign-up handoff, legal pages, es-419 localization | Sign-in handoff needs Phase 5; rest can run in parallel with 4/5 |

Mobile app and IoT firmware continue to evolve but are not the focus of this program.

---

## 2. Repository strategy

| Repo | Purpose |
|---|---|
| `tti-helper-aws` (existing) | All back-end. Multi-stack CDK: `IotCoreStack` (existing), `AuthStack`, `AdminApiStack`, `ClientApiStack`, `SharedDataStack`, `OtaStack`. |
| `tti-helper-mobile` (existing) | Flutter app. **Two build flavors** (added Phase 5): `mobile` (full feature, US + LatAm — Troubleshooting + Inspection + ACK/SILENCE/RESET) and `monitor` (Monitoring product — ACK only, no Inspection, alarm feed rendered as Monitoring + Reporting; adds Flutter-web build for the web Monitoring surface). Same codebase, two store listings. **No new `tti-helper-monitor` repo.** |
| `tti-helper-iot` (existing) | Firmware. |
| `tti-helper-web` (existing) | Public marketing site / company front door at `https://ttihelper.com`. Static HTML/CSS in `public/`, deployed via Cloudflare Workers + Assets on push to `main`. |
| `tti-helper-portal-admin` (new) | Internal back-office console. English only. Private hosting (CloudFront + WAF IP-allowlist). |
| `tti-helper-portal-client` (new) | Multi-tenant customer portal. i18n from day one. Public hosting. |

**Why two portal repos, not a monorepo:** admin and client are *different products* — different users, deploy cadence, security posture, and design investment. Keeping them separate prevents accidental cross-contamination (e.g., tenant-scoping bugs, internal utilities leaking to the client bundle).

**Shared code between portals:** auto-generate a TypeScript API client from an OpenAPI spec in `tti-helper-aws`, publish to a private registry (GitHub Packages or AWS CodeArtifact), consume from both portals. No shared-component monorepo.

---

## 3. Cross-cutting decisions (apply across all phases)

### 3.1 Multi-tenancy model — pool

**Decision: pool model** — shared tables with `tenant_id` partitioning.

#### Non-negotiable rules

1. **`tenant_id` is a partition key, always.** DynamoDB: `PK = tenant_id#<entity>` or composite `(tenant_id, entity_id)`. No table multi-tenant data without `tenant_id` in the key.
2. **Tenant scoping lives in the data-access layer, not in handlers.** Repository pattern: `TenantScopedRepo(tenantId)` injects `tenant_id` into every read/write. Handlers never touch raw queries.
3. **`tenant_id` is a JWT claim, not a request parameter.** Cognito pre-token-generation Lambda injects `tenant_id` into the ID token from a `Users` table lookup. API Gateway authorizer extracts it. Never trust a `tenant_id` sent in a request body.

#### Things to build in from the start

- **IoT topic scoping**: topic scheme becomes `dt/esp32/ttireaderv1/<tenant_id>/<thingID>` so the IoT policy can enforce per-tenant MQTT isolation. Devices get provisioned with a tenant-scoped cert.
- **Per-tenant API Gateway usage plans**: rate + quota limits per tenant to contain noisy neighbors.
- **Tenant-isolation test harness**: a CI test that, for every API endpoint, asserts "as tenant A, cannot see tenant B's rows". Single most effective defense against tenant-leak bugs.
- **Silo escape hatch**: design data access so one whale tenant can later move to an isolated stack via data export + `tenant_id` rewrite, without code changes.

#### Admin portal differences

Admin queries explicitly opt out of tenant scoping via an `UnscopedRepo()` class, only instantiated in the admin-API Lambda. Never in client-API code.

### 3.2 Regions & markets — v1

#### Launch scope

| Area | Decision |
|---|---|
| AWS region | `us-east-2` only |
| Markets | US + Latin America **excluding Brazil** (Mexico, Argentina, Chile, Colombia, Peru, Ecuador, Venezuela, etc.) |
| Excluded markets | Brazil, Canada, EU, UK, Middle East, Africa, Asia |

#### Accepted tradeoff

LATAM users (Argentina, Chile, Peru, etc.) will see ~150–200ms RTT to `us-east-2`. Acceptable for device management and alarm feed in v1. Add `sa-east-1` later when any of the triggers below fires.

#### Triggers for adding a second region

- First signed LATAM customer on an enterprise-tier SLA
- First customer complaint about alarm latency with supporting evidence
- Reaching 10+ paying LATAM tenants (arbitrary threshold — revise when approached)
- Any customer in a country with data-residency law that forbids US storage

#### Cheap insurance for future regions (build in from day one)

- Tenant record has a `region` attribute, always `us-east-2` in v1. Adding a region later = data migration, not schema change.
- Mobile app reads its AWS config (endpoint, Cognito pool, region) from the tenant payload at login. Hardcoded in v1 but the code path exists.
- CDK stacks accept region as a parameter even when we deploy only one.

### 3.3 Internationalization (i18n)

#### Languages for v1

| Language | Coverage | Priority |
|---|---|---|
| English | US, internal staff | P0 |
| Spanish (es-419, LATAM neutral) | Mexico, Chile, Argentina, Colombia, Peru, etc. | P0 |

- **Admin portal (Phase 4):** English only.
- **Client portal (Phase 5) + mobile app + marketing site (Phase 6):** English + Spanish.

**Deferred:** Portuguese (no Brazil), French (no Canada), RTL languages (no Middle East).

#### i18n infrastructure (non-negotiable from day one)

Retrofit after the fact is much more expensive than up-front setup.

1. **Externalized strings.** Zero hardcoded user-facing text. Catalogs in `.arb` (Flutter) and JSON (web).
2. **Parameterized messages.** ICU MessageFormat for plurals/gender. Never string-concat sentences.
3. **Locale-aware formatting.** Dates, numbers, currency via locale-aware APIs (`intl` in Flutter, `Intl` on web). UTC in storage, local display.
4. **Locale detection & override.** Device locale at startup, English fallback, user-override persisted.

#### Stack choices

- **Mobile (Flutter):** `flutter_localizations` + `intl` + `.arb` files. `flutter gen-l10n` for typed accessors.
- **Client portal (web):** pick one during design — `next-intl` (if Next.js), `react-intl`, or `i18next`. JSON catalogs per locale.
- **Marketing site (`tti-helper-web`):** lighter-weight approach acceptable since it's static HTML — could be per-locale `public/es/` subfolder or a build step that emits both. Decide during Phase 6.
- **Back-end:** error *codes* over the wire, not error messages. Portals translate codes.

### 3.4 Compliance baseline

#### In scope for v1

- **US — CCPA / CPRA** (California) plus growing state patchwork (Texas, Virginia, Colorado, etc.)
- **Mexico — LFPDPPP**
- **LATAM general** — Argentina PDPL, Chile Law 19.628 (modernization in flight), Peru, Colombia, etc. All GDPR-adjacent in spirit.

#### Practical target

Build a **"LATAM GDPR-lite" baseline** that satisfies ~90% of LATAM + US requirements:

- Consent capture at signup and on material changes
- Data-export endpoint (self-service in client portal)
- Data-erasure endpoint (self-service with hard-delete workflow)
- Audit log of admin access to tenant data
- Privacy policy, cookie notice on client portal **and on `tti-helper-web`**
- DPA (Data Processing Agreement) template for enterprise customers
- Data Processing Inventory — sub-processors (Cognito, SES, SNS, S3, DynamoDB, **Cloudflare for the marketing site**) documented

Clean foundation for adding EU (GDPR) or Brazil (LGPD) later.

#### Out of scope for v1

- GDPR (no EU users)
- LGPD (no Brazil)
- PIPEDA / Quebec Law 25 / Bill 96 (no Canada)
- Country-specific data-localization laws (Russia, China, Saudi, India)

---

## 4. Phase 3 — `tti-helper-aws` (back-end foundation)

### CDK stack decomposition (all in `us-east-2`)

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

### Phase 3 deliverables checklist

- [ ] All six CDK stacks defined and deployable to `us-east-2` / `prod`
- [ ] Pre-token-generation Lambda injecting `tenant_id` claim
- [ ] OpenAPI spec covering admin + client endpoints
- [ ] Generated TypeScript client published to private registry
- [ ] Tenant-isolation CI test harness passing
- [ ] IoT topic scheme migrated to `dt/esp32/ttireaderv1/<tenant_id>/<thingID>` (with mobile + firmware updates coordinated)

---

## 5. Phase 4 — `tti-helper-portal-admin` (internal console)

| Aspect | Decision |
|---|---|
| Users | TTI Helper internal staff (~5) |
| Auth | Admin Cognito pool (from Phase 3 `AuthStack`) |
| i18n | English only |
| Hosting | CloudFront + WAF IP-allowlist (or VPN — see open question D) |
| Data access | `UnscopedRepo()` — explicit opt-out of tenant scoping; never used in client-API code |

### Capabilities (initial)

- Tenant management: create / suspend / delete tenants
- User management: provision client users, reset MFA, audit login activity
- Device fleet view: all devices across all tenants, OTA orchestration
- Audit log viewer: every admin action against tenant data
- Firmware management: upload, sign, publish via IoT Jobs

### Why Phase 4 before Phase 5

Until TTI staff can provision a tenant, no client can sign in. Self-service signup on the client portal is **not** part of v1 (see project memory: mobile app is login-only; portal will follow the same model initially).

---

## 6. Phase 5 — `tti-helper-portal-client` (customer SaaS)

| Aspect | Decision |
|---|---|
| Users | Customer companies, multi-tenant |
| Auth | Client Cognito pool (from Phase 3 `AuthStack`), `tenant_id` JWT claim |
| i18n | English + Spanish (es-419), infrastructure mandatory from day one |
| Hosting | Public (CloudFront or similar) |
| Data access | `TenantScopedRepo(tenantId)` — `tenant_id` from JWT, never from request body |

### Module composition (revised 2026-04-26)

`tti-helper-portal-client` carries **only the Tech Management (TM) module**, served identically to US and LatAm tenants. Monitoring is **not** a TS module in portal-client — it's delivered as a second Flutter build flavor of `tti-helper-mobile` (see "Monitoring delivery" below).

| Region | Surfaces available |
|---|---|
| US (and other regulated markets: CA, most EU) | TM module in `tti-helper-portal-client` (web) + `mobile` flavor of `tti-helper-mobile` |
| LatAm (and other unregulated markets) | Same as US **plus** `monitor` flavor of `tti-helper-mobile` (mobile + Flutter web) for the Monitoring product |

**Why asymmetric:** US fire-alarm monitoring is regulated (UL 827, NFPA 72). Shipping Monitoring only in unregulated markets keeps TTI out of central-station certification scope. Tech Management has no such restriction, so it ships universally. Monitoring = in-house self-monitoring by the client's own staff, not a third-party central-station service; ToS will say "best-effort notification, not a life-safety service."

**TM codebase (confirmed 2026-04-26):** TM is a single shared module in `tti-helper-portal-client` — same code rendered identically for US and LatAm tenants. No parallel TM implementation.

**Per-device gating for Monitoring (confirmed 2026-04-26):** Monitoring eligibility is gated at the **site / device level**, not tenant level. Each device/site record carries its own region/country attribute. A Mexican-headquartered tenant with a US-installed device cannot use Monitoring on that device; only LatAm-located devices appear in Monitoring views. Data-model implication: device records carry a region attribute; Monitoring queries filter by per-device eligibility, not tenant HQ region.

### Monitoring delivery — `tti-helper-monitor` build flavor (confirmed 2026-04-26)

The Monitoring user experience ships as a **second Flutter build flavor of `tti-helper-mobile`**, branded `tti-helper-monitor`. Same codebase, different feature set, separate app store listings.

| Aspect | Decision |
|---|---|
| Repo | `tti-helper-mobile` — **no new repo for monitor** |
| Build flavors | `mobile` (full feature) + `monitor` (Monitoring-restricted) |
| Build targets | iOS (native), Android (native), Flutter Web. **Desktop deferred** to Phase 5 design. |
| Distribution | **Global** — all stores + a single global URL for the web build. Eligibility enforced server-side (tenant region + per-device region attribute). |

**Per-flavor scope:**

| Capability | `mobile` flavor | `monitor` flavor |
|---|---|---|
| FACP commands | ACK + SILENCE + RESET | **ACK only** (RELAY + UART transports) |
| Device commands | WIFI, SETUP_UART, OTA | WIFI, SETUP_UART, OTA |
| Inspection module (NFPA PDFs) | ✅ | ❌ |
| Live alarm feed | Troubleshooting screen (real-time + commands) | Monitoring screen (history-style + Reporting) |
| Device Management screen | ✅ | ✅ |
| Settings / Configuration UI | ✅ | ✅ |
| BLE provisioning | ✅ (mobile builds) | ✅ (mobile builds only — Web Bluetooth has no Safari/iOS support) |

**Web Monitoring is the Flutter-web build of the monitor flavor.** No parallel TS Monitoring module in `tti-helper-portal-client`. One product spec across mobile + web surfaces.

**IAM enforcement caveat:** AWS IoT IAM is topic-based, not body-based — a monitor-flavor user could in principle craft a SILENCE/RESET payload to the same MQTT topic. Enforcement at MVP is UI-only (monitor flavor doesn't render those buttons). If the threat model later requires it, add a Lambda IoT rule that drops disallowed commands published from monitor-flavor identities.

### Provisioning workflow (3 stages, 3 surfaces)

| Stage | Surface | Actor | What |
|---|---|---|---|
| 1. Device enrollment | `tti-helper-portal-admin` | TTI Helper staff | Bind a physical ESP32 to a tenant; issue tenant-scoped cert; set the device's region attribute |
| 2. Physical install | On site | TTI tech **or** technical partner | Wire ESP32 to the panel; BLE-push Wi-Fi credentials; device boots and connects to AWS IoT with its tenant cert |
| 3. Role assignment | `tti-helper-portal-client` (TM module) | Tenant admin / privileged user | Assign alert-recipient roles to tenant users — defines who receives Monitoring alerts |

Runtime fan-out: ESP32 publishes once to its tenant-scoped MQTT topic; multiple `tti-helper-monitor` instances (any platform, any user) subscribed to that topic all receive the alarm via standard MQTT pub/sub.

### Distribution and gating — option (b) global with server-side gating (confirmed 2026-04-26)

- Mobile builds (`mobile` and `monitor`) are listed globally on iOS App Store and Google Play.
- Flutter-web build of monitor served at one global URL.
- Sign-in succeeds globally; per-device eligibility filter then determines what each user sees and can act on.
- Empty-state UX: a US-tenant user signing into the monitor flavor (no eligible devices) sees a graceful "no devices" screen with a help link, not a sign-in failure.
- **Fallback:** If app-store review later flags the monitor app as US-unsuitable life-safety product, retreat to (a) — region-restricted store listings + LatAm-targeted web URL. (b) preserves both options; (a) doesn't.

~13 open product decisions are still parked on this split (planning 2026-04-22). #1 (mobile in LatAm) and #2 (gating granularity) resolved 2026-04-26. Resolve the rest before Phase 5 design.

### Onboarding flow (v1)

Pre-portal: tenant + initial user are created by TTI staff via the admin portal (Phase 4) or, until then, via the existing admin CLI scripts at `tti-helper-aws/scripts/create-user.sh`. The client portal does **not** include self-service signup in v1.

---

## 7. Phase 6 — `tti-helper-web` (public marketing site)

The site already exists at `https://ttihelper.com` — Phase 6 is **scoped updates**, not a build-from-zero.

### Phase 6 work items

1. **Sign-in / sign-up entry points** that hand off to the client portal (Phase 5). Sign-up may simply route to a "request access" form in v1, given there's no self-service signup yet.
2. **Legal / privacy pages** aligned with the §3.4 compliance baseline: privacy policy, cookie notice, sub-processor list, DPA download for enterprise prospects.
3. **Spanish (es-419) localization** to match the client portal.
4. **Cookie consent banner** if analytics or marketing pixels are added.

### Out of scope for Phase 6

- Customer-authenticated content (lives in Phase 5)
- Documentation portal (separate decision; could be a `/docs` subroute later)

### Sequencing note

Phase 6 sign-in handoff depends on Phase 5 endpoints being live. The legal-pages and i18n work items can run in parallel with Phases 4/5.

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
| 12 | Phasing | 3 = aws, 4 = admin, 5 = client, 6 = web (subject to change) |

---

## 9. Open questions (still to resolve before design)

| # | Question | Affects phase | Why it matters |
|---|---|---|---|
| A | Tenant-to-region binding mechanics | 3 | Even though v1 is single region, decide now whether tenants pick at signup or we auto-assign. |
| B | Firmware region binding | 3 | Single binary with runtime region discovery, or per-region firmware? Keeping single-binary is simpler if we later go multi-region. |
| C | OTP channel for LATAM users | 3 | SMS via SNS (expensive per message in LATAM) vs email via SES. Probably email-first with SMS opt-in. |
| D | Admin portal hosting | 4 | CloudFront + WAF IP-allowlist, or put it behind a VPN? IP-allowlist is simpler. |
| E | OpenAPI toolchain | 3 | Speccy/Stoplight/TypeSpec? Decide once. Output: TypeScript client + Python server stubs. |
| F | Billing model | 5 | Needed for client-portal signup flow. Out of Phase 3–6 scope? |
| G | Offline tolerance for mobile in LATAM | 3 | Confirm MQTT retry and queue behavior before launch. |
| H | Enterprise contract language | 5/6 | Some LATAM jurisdictions require Spanish for consumer/enterprise contracts. Legal review before first enterprise deal. |
| I | LatAm Monitoring vs Tech Management module split | 5 | ~13 parked product decisions (memory: 2026-04-22). Resolve before Phase 5 design. |
| J | `tti-helper-web` i18n approach | 6 | Per-locale subfolder, build-step, or runtime swap? Decide during Phase 6 design. |

---

## 10. Cross-repo impact when phases ship

| Phase | What also has to change |
|---|---|
| Phase 3 (aws) | `tti-helper-iot` — IoT topic scheme gains `<tenant_id>`; `tti-helper-mobile` — subscriptions and config payload from new tenant lookup |
| Phase 4 (admin) | None outside the new repo + back-end policy updates |
| Phase 5 (client) | `tti-helper-web` — sign-in link target points at the new portal |
| Phase 6 (web) | None back-end; may surface new `Contact` / `Request Access` form that drops into a queue or email — decide during design |

---

## 11. What must happen before we start Phase 3

1. **E2E test Phase 2** (inspection feature) on real hardware and real client scenarios. Current top priority.
2. **SQS message buffer (#22 in `ROADMAP.md`)** — blocking dependency for Phase 3. No point building a portal if Phase 2 is lossy.
3. **Resolve the open questions above** — at minimum A, B, C, E before any CDK or portal code lands.
4. **Confirm phase sequencing** — the 3 → 4 → 5 → 6 order is a proposal; revisit when Phase 2 testing wraps.

Until those gates are cleared, Phases 3–6 are planning only.
