# TTI Helper — Cross-Repo Roadmap

Durable phase-level plan across all TTI Helper repos. Consolidated from prior mobile-session memory, 2026-04-19.

Status key: ✅ Done | 🔄 In Progress | ⏳ Pending

`TODO.md` tracks the immediate next-steps. This file tracks the full phase plan.

---

## Phase 1 — Troubleshooting (`tti-helper-mobile`)

### Done ✅
- Welcome screen (3 options, Troubleshooting active)
- Session init + management (create, close, history, location prompt)
- Device comm settings (baud, data/stop bits, parity)
- FACP model system + FireLite 9050 parser
- MQTT alarms + commands
- Cognito email OTP auth
- Biometric lock
- BLE provisioning + WiFi update via BLE (physical-device test confirmed)
- Ground fault wizard + loop topology
- PDF session export (3 entry points)
- **#U1 Theme rebrand** — landed on teal-green primary (`#00A884`) + light-gray/off-white surface in `lib/shared/theme/app_theme.dart`; adaptive light/dark modes.

### Pending ⏳
- **#8 SMS OTP** — code deployed; blocked on AWS toll-free number. Console → SNS → Origination identities → Request toll-free; then sandbox-verify `+17863570769`.
- **#P1 FACP parsers** — additional models from manuals: Notifier NFS2-3030, Silent Knight SK-5208/5820XL, Simplex 4100ES/4010ES, Gamewell-FCI E3, Siemens Cerberus PRO/Sinteso, Bosch FPA-5000, Mircom FX-2000, Fire-Lite MS-9200UDLS, Kidde/Honeywell.

### Business / Legal
- **#L1** Form company — Delaware C-Corp recommended. ⏳
- **#L2** Domain — `ttihelper.com` purchased at Cloudflare Registrar (2026-04-20); DNS + `no-reply@ttihelper.com` sending via SES. ✅
- **#L3** Apple Developer account ($99/yr) at developer.apple.com. ✅
- **#L4** IoT SIM contract — consider Hologram or Twilio Super SIM for early stage; re-engage AT&T/Verizon once volume justifies. ⏳

---

## Phase 2 — Inspection (`tti-helper-mobile`)

### Done ✅
- Inspection button active on Welcome Screen
- Zone checklist
- Photo capture per zone
- Inspector signature pad
- Inspection report PDF

### Infrastructure — Pending ⏳ (MUST precede Phase 3)

**#22 SQS Message Buffer** — prevents lost FACP events when mobile is offline.

```
IoT Device → AWS IoT Core → IoT Rule → SQS FIFO → Lambda → DynamoDB
                                                                 ↓
                                         Mobile ← API Gateway ←──┘ (on reconnect)
```

- **AWS side** (`tti-helper-aws`): IoT Core Rule → SQS FIFO; Lambda consumer → DynamoDB; API Gateway fetch endpoint.
- **Mobile side**: on MQTT reconnect, fetch events since `last_seen`; merge into local SQLite (dedup by timestamp + thingId).
- Cost: SQS/Lambda/DynamoDB/API Gateway all within free tier at current scale.

---

## Phase 3 — Portal Admin + Portal Client (`tti-helper-aws` CDK additions + two new portal repos)

> **See `PHASE_3_PLAN.md`** for the full planning doc — multi-tenancy model, regions, i18n, compliance, stack decomposition, decision log, and open questions. The sections below are the older feature list, kept for history; the plan doc is now the source of truth for Phase 3 strategy.

### Tech stack
| Layer | Technology |
|-------|-----------|
| Frontend | Next.js (React) |
| Backend | Python Lambdas |
| Infra | AWS CDK additions in `tti-helper-aws` |
| Auth | 2nd Cognito User Pool (admin + client, separate from mobile pool) |

Portal → API Gateway → Python Lambda → AWS IoT Core Management API (not MQTT). OTA uses AWS IoT Jobs + S3.

### Features ⏳
| # | Task | Notes |
|---|------|-------|
| 3.0 | Company website (`ttihelper.com`) | Public marketing site, launched with portal |
| 3.1 | Admin sign-in | Cognito User Pool: TTI Admins |
| 3.2 | Client sign-in | Cognito User Pool: Clients |
| 3.3 | Client enrollment | Admin creates client record (company, contacts) |
| 3.4 | Device assignment | Assign IoT Thing (by MAC/ID) to client and/or employee |
| 3.5 | OTA firmware | Upload to S3 → create AWS IoT Job → track per-device status |
| 3.6 | Certificate rotation | Per-device cert renewal via IoT Jobs |

### AWS CDK additions (`tti-helper-aws`) ⏳
- IoT Jobs
- S3 firmware bucket
- DynamoDB tables (clients, device assignments, job status)
- API Gateway (REST or HTTP API)
- 2nd Cognito User Pool (admin + client portal)

### Hardening ⏳
- **IoT Policy** — replace the wildcard `TTIReaderPolicy` (`iot:*` on `*`) currently attached to all 5 Things with the per-device scoped policy already defined in `tti-helper-aws/lib/tti-helper-aws-stack.ts` (~L204–210):
  - `iot:Connect` on `client/${iot:ClientId}`
  - `iot:Publish` on `dt/esp32/ttireaderv1/${iot:ClientId}`
  - `iot:Subscribe` + `iot:Receive` on `cmd/esp32/ttireaderv1/${iot:ClientId}`

### Pending AWS actions — #21 Production OTP readiness (before go-live) ⏳
- **SES:** Console → SES → Account dashboard → Request production access (send to any email)
- **SNS:** Console → SNS → Text messaging → Exit sandbox (SMS to any phone)

### Execution order
1. Create `tti-helper-portal` repo
2. Land CDK additions in `tti-helper-aws`
3. Build portal features 3.0 → 3.6
4. Harden IoT policy before go-live

---

## Phase 4 — Record of Completion (`tti-helper-mobile`)

⏳ Pending — all items.

- Wizard filled by technician after a troubleshooting session
- Fields: work performed, parts replaced, technician info, sign-off
- Generates PDF completion certificate (reuses session PDF pipeline)
- Activates the greyed-out button on the Welcome Screen

---

## Emergency Fleet OTA (urgent + mandatory updates)

Fleet-wide firmware push for exceptional cases — security CVE, alarm-path
bug with life-safety impact, certificate rotation on a tight window,
compliance deadline. The existing per-device OTA flow (mobile publishes
`OTA` to `cmd/esp32/ttireaderv1/{thingID}`) is not designed for this.

### Prerequisites (do these regardless of delivery mechanism)

1. **Signed firmware** — device verifies an ECDSA signature against a
   public key burned into firmware before flashing. Without this, anyone
   who can publish MQTT can push arbitrary code. Largest single gap in
   the current OTA design. **Repo:** `tti-helper-iot`.
2. **A/B OTA partitions + rollback** — use ESP-IDF `esp_ota_*` so a
   failed-boot image auto-reverts via bootloader. Verify this is on;
   if not, wire it in. **Repo:** `tti-helper-iot`.
3. **`mandatory: true` flag in OTA payload** — when present, firmware
   skips any user confirmation and flashes immediately. Non-mandatory
   updates can still be deferred. **Repos:** `tti-helper-iot`,
   `tti-helper-mobile`.

### Delivery mechanism — three options

**Option A — AWS IoT Jobs (recommended at fleet > ~100 devices)**
- Admin creates a Job via `aws iot create-job` targeting a thing group.
- Firmware-side: subscribe to `$aws/things/{thingName}/jobs/notify-next`;
  handle `GetPendingJobExecutions` / `UpdateJobExecution`.
- Policy update: add `iot:Subscribe|Receive|Publish` on
  `$aws/things/${iot:ClientId}/jobs/*` to `tti-iot-device-policy`.
- Benefits: staged rollout rate, per-device status (`QUEUED → IN_PROGRESS
  → SUCCEEDED / FAILED / TIMED_OUT`), abort-on-failure-rate, offline
  durability (queued on the thing), timeouts, audit.
- Cost: ~$0.0025 per device per execution (negligible).
- Work: ~300–500 LOC of C on firmware + CDK / admin CLI on cloud.
- **Bootstrapping:** devices need to be reflashed once to install the
  Jobs agent. Use the existing `OTA` command path to do that push.

**Option B — Broadcast MQTT topic (fast to ship, do not use alone)**
- Add `cmd/esp32/ttireaderv1/broadcast/ota`; all devices subscribe.
- Arrives in <1s to online devices; offline devices miss it unless the
  publish is retained (and retained on wildcard topics creates re-flash
  loops on reconnect). No per-device ack. No rollback. No staged rollout.
- Acceptable only layered behind signed firmware + A/B partitions + a
  cancel/override mechanism — which is most of what Jobs gives you for
  free. Mostly useful as a supplementary "notify" channel, not the
  primary delivery path.

**Option C — Brute-force loop over existing per-device OTA (stopgap)**
- Admin CLI (new script under `tti-helper-aws/scripts/`) iterates every
  `thingID`, publishes the existing `OTA` payload to
  `cmd/esp32/ttireaderv1/{thingID}`, rate-limited (e.g. 5/sec).
- Zero firmware change — uses tested code.
- Status arrives via existing `dt/esp32/ttireaderv1/{thingID}/status`
  topic after device reboots with new `firmware_version`; script watches
  and reports laggards.
- No abort safety net mid-flight beyond stopping the script. Offline
  devices need a rerun.
- Fit: good emergency button for small fleet, exceptional use.

### Recommended execution order

1. **Prerequisites first** (signed firmware, A/B partitions, mandatory
   flag). Dominates any fleet-safety conversation; do before either
   delivery path.
2. **Option C as near-term emergency button.** Ship a rate-limited admin
   script once prerequisites are in. Covers the "urgent exception" use
   case today with zero new infrastructure.
3. **Option A when fleet reaches ~100 devices or Phase 3 lands** (item
   3.5 already references IoT Jobs). At that scale the script's lack of
   abort / observability starts to bite.
4. **Skip Option B** as a primary path. Dangerous for a life-safety
   product.

---

## Multi-Carrier LTE / Runtime APN Configuration

Today the firmware has a single compile-time APN (`tti-helper-iot/src/ttireader_modem.cpp:19` — `m2m.com.attz` = AT&T M2M). Switching carriers means editing the constant, rebuilding, and reflashing every device. Prerequisite for the LatAm expansion documented in `memory/project_latam_monitoring_product.md` since each target country typically means a different SIM/carrier (Telcel/Movistar/Claro in MX, Entel/Movistar/WOM in CL, Vivo/Claro/Tim in BR, etc.). One test already exists for Entel Chile (`imovil.entelpcs.cl`, commented-out line at the same location).

### Design options

**Option A — BLE provisioning accepts APN (MVP)**
- Extend the BLE protocol (currently `A:<pin>`, `S:<ssid>`, `P:<password>`) to also accept:
  - `G:<apn>` — APN string
  - `GU:<user>`, `GP:<password>` — optional GPRS credentials
- Store in NVS like WiFi creds; modem reads APN from NVS on connect, falls back to compile-time default if empty.
- Mobile "Provision WiFi" sheet grows an optional APN field.
- ~50 LOC firmware + small mobile change.

**Option B — MQTT `SETUP_APN` command (companion to A)**
- New command on `cmd/esp32/ttireaderv1/{thingID}` (same pattern as existing `SETUP_UART`):
  ```json
  { "command": { "name": "SETUP_APN", "apn": "imovil.entelpcs.cl" } }
  ```
- Writes NVS, reconnects modem. Lets you fix "wrong SIM in that device" without a site visit.
- Chicken-and-egg: only works when device is already online, so always a companion to A, not a replacement.

**Option C — IMSI-based auto-detection (polish for scale)**
- On boot, read IMSI via `sim_modem.getIMSI()`; leading digits encode MCC+MNC (country + carrier).
- Match against a small in-firmware table (AT&T US, Entel CL, Telcel MX, Hologram, Twilio Super SIM, ...) → auto-select APN.
- Fall back to NVS (set by A/B) → fall back to compile-time default.
- ~100 LOC + an IMSI table that grows as you enter new markets.
- Only worth it once per-device APN entry becomes operational pain (3+ carriers in the field).

### Recommended execution order

1. **Phase 1** — ship Options A + B together. Covers every current and near-term multi-carrier need (US AT&T M2M + Chile Entel test + any new LatAm country via reprovisioning). Trivial to do them together since B reuses A's NVS handler.
2. **Phase 2** — add Option C once the fleet spans 3+ carriers and installers need a zero-config workflow. Defer until real operational pain justifies the table-maintenance burden.

### Hardware caveat (purchasing, not firmware)

Multi-carrier ≠ multi-region in hardware. LilyGO-T-SIM7600 comes in regional variants (NA / EU / LA / G-global) with different LTE band support. For one firmware + one SKU across US and LatAm, procure the **SIM7600G (global)** variant going forward. Confirm which variant the currently-deployed units carry before assuming they'll roam internationally.

### Doc drift to clean up when touching this

- `tti-helper-iot/doc/PROJECT_OVERVIEW.md` still references "SIM800" in a couple of places. Code is on SIM7600 (`TINY_GSM_MODEM_SIM7600`). Fix when editing modem code.

---

## Cost context

Phase 1 monthly total: ~$0.25–$1. Main drivers at scale:
1. SNS SMS OTP (200 techs ≈ $40/mo); consider email-first.
2. IoT Core message volume (panel spam during drills).
3. SNS toll-free leasing fee (~$2/mo).
4. Phase 3 additions — DynamoDB + API Gateway + S3 + 2nd Cognito pool (low at current scale).

Full breakdown lives in mobile-session memory (`aws_costs.md`).

---

## Cross-repo gating order

```
Phase 1 (mobile)  ─────────────► in progress, a few pending items
Phase 2 (mobile)  ─────────────► features done; #22 SQS buffer blocks Phase 3
Phase 3 (portal + aws) ────────► starts after #22 lands
Phase 4 (mobile)  ─────────────► independent; can start any time after Phase 1
```
