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
- BLE provisioning + WiFi update via BLE
- Ground fault wizard + loop topology
- PDF session export (3 entry points)

### Pending ⏳
- **#8 SMS OTP** — code deployed; blocked on AWS toll-free number. Console → SNS → Origination identities → Request toll-free; then sandbox-verify `+17863570769`.
- **#20 BLE WiFi update** 🔄 — code done (commit `fe1cb26`), needs physical-device test.
- **#U1 Rebrand colors** (red/white) — palette in `phase1_pending.md`; change confined to `lib/shared/theme/app_theme.dart` plus audit of hardcoded `Color(0xFF00A884)`.
- **#P1 FACP parsers** — additional models from manuals: Notifier NFS2-3030, Silent Knight SK-5208/5820XL, Simplex 4100ES/4010ES, Gamewell-FCI E3, Siemens Cerberus PRO/Sinteso, Bosch FPA-5000, Mircom FX-2000, Fire-Lite MS-9200UDLS, Kidde/Honeywell.

### Business / Legal ⏳
- **#L1** Form company — Delaware C-Corp recommended.
- **#L2** Domains — `ttifire.com` + `ttifire.io` via Cloudflare (confirm purchased/pointed).
- **#L3** Apple Developer account ($99/yr) at developer.apple.com.
- **#L4** IoT SIM contract — consider Hologram or Twilio Super SIM for early stage; re-engage AT&T/Verizon once volume justifies.

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
| 3.0 | Company website (`ttifire.com`) | Public marketing site, launched with portal |
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
