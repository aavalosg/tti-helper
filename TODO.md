# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(16) — Notifier parser manual-driven rebuild

**Local state as of 2026-05-01:**
- `pubspec.yaml` is `1.0.0+15` (will bump to `1.0.0+16` once the parser rewrite + tests pass).
- HEAD on `tti-helper-mobile/main` is `8303fd2`, **2 commits unpushed** to origin (`5c13b38` icons + `8303fd2` bump from (15)). Push at next mobile commit.
- (15) released and field-tested 2026-05-01 — launcher icon (bell+gear+check on green) confirmed on physical device. No functional regression vs (14).

**Plan for (16):** Manual-driven rewrite of the Notifier FACP parser. Replaces the bench-calibrated `notify360_parser.dart` (single 2026-04-03 calibration session, gaps in Pre-Alarm / Disabled Points / Fire Control / CO Alarm / MN family / structured address+zone extraction / multi-line trouble form). Mirrors the EST iO refactor from (13). User services NFS-320, NFS-640, AND NFS2-640 panels — needs all three covered. "notify360" is not a real Notifier product (typo or shorthand from the 2026-04-03 session that stuck in the code).

Manual sources in `tti-helper-mobile/facp_manual/Notifier/`:
- `NFS-320-E-Operation Manual.pdf` (doc 52747, 2011) — canonical for NFS-320 family
- `NFS2-640-E-Operation Manual.pdf` (doc 52743, 2013) — canonical for NFS2-640 family
- Installation manuals (52745 / 52741) — secondary
- `NFS-640-Programming Manual.pdf` (doc 51333, 2003, legacy) — set aside; `notifierNfs640` enum still routes to the same parser (DPI-232 interface 51499 is shared across the family)

Format-identity finding (side-by-side compare 2026-05-01): NFS-320 and NFS2-640 use **identical** event format (banners, type codes, address/zone, time/date, multi-line trouble form). 640-only additions are purely additive — no banner conflicts. **One `NotifierParser` covers all three.**

**Locked decisions:**
- Architecture: one `NotifierParser`, model-agnostic core.
- `FacpModel` enum: replace `notify360` with three values — `notifierNfs320`, `notifierNfs640`, `notifierNfs2640` — all dispatching to the same parser. Display labels: `'Notifier NFS-320'` / `'Notifier NFS-640'` / `'Notifier NFS2-640'` (matches existing brand+model convention).
- File rename: `lib/core/facp/parsers/notify360_parser.dart` → `notifier_parser.dart`; `Notify360Parser` → `NotifierParser`.
- SharedPrefs migration: B-default-modern — `notify360` → `notifierNfs2640` silently on first launch (one-shot, idempotent; mirrors the `estIO1000` → `estIO500` migration from (13)).
- Bench coverage: user will verify on NFS-320, NFS-640, AND NFS2-640 panels (multiple clients across all three).

**7-step execution plan:**
1. **Write the protocol reference doc** — new `tti-helper-mobile/doc/notifier_protocol.md`. Captures the 11 status banners (`SYSTEM NORMAL` · `ALARM:` · `TROUBL` · `ACTIVE` · `ACTIVE SECURITY` · `ACTIVE FIRE CONTROL` · `ACTIVE TAMPER` · `PREALM` · `DISABL` · `ACKNOWLEDGE` · `SIGNALS SILENCED`); Fire / Security / Supervisory / CO / Non-Alarm / MN type code tables; address formats (`MNNN`, `DNNN`, `BNN`, multi-line `MNNN ZNN OPEN CIRCUIT`); zone (`ZNNN`); time/date (`HH:MMa/p MMDDYY`); Pre-Alarm percentage (`055%/4`); 320-vs-640 deltas (MN family, DVC troubles, MNS outputs, paging codes). Standalone commit before any parser code.
2. **Update `FacpModel` enum** in `lib/core/facp/facp_model.dart`. Remove `notify360`. Add `notifierNfs320`, `notifierNfs640`, `notifierNfs2640` with the display strings above. Keep `expectsPairs` and `descriptionFirst` returning `false`.
3. **Rewrite the parser** — rename file/class, replace body with manual-driven implementation: banner-based event classification, structured type code / zone / address extraction, multi-line trouble support, MN / Pre-Alarm / Disabled / Fire-Control handling, CO Alarm differentiation. Keep existing `FacpEvent` model.
4. **Update parser dispatcher** — wherever `FacpModel` is switched onto a parser (likely `facp_parser.dart` or alarm/event provider), route all three new enum values to the same `NotifierParser` instance. Remove the `notify360` branch.
5. **SharedPrefs migration** — mirror the (13) `estIO1000` → `estIO500` migration (`fd238e0`). Marker key `facp_model_migration_v16_notifier_rename`. Idempotent.
6. **Tests** — port existing `notify360_parser` tests to `notifier_parser_test.dart`. Add coverage for each banner, each event family, each address format, multi-line trouble, MN events, Pre-Alarm percentage, CO alarm. Maintain 171+ passing.
7. **Build, bench-verify, ship** — bump pubspec to `1.0.0+16`, `flutter build ipa`, upload via Transporter, install + bench-verify on NFS-320 / NFS-640 / NFS2-640 panels, External submission with "What to Test" notes per the standing rule.

**Build history on App Store Connect (recent):**
- (10) — History-page event ordering under panel burst; firmware v1.0.7 OTA companion (UART 256 B → 4 KB drain-per-tick). Field-tested.
- (11) — Hybrid EST iO pairing (wrong direction — superseded by (12)).
- (12) — EST iO pairing direction corrected; `descriptionFirst => false` for all models. Field-tested 2026-04-29.
- (13) — Bundled: `_classify()` Battery Missing/Power Loss/etc descriptor pairing + full EST iO parser rewrite (17 STATUS codes, 4 address formats, iO64/iO500 + iO64/iO1000 split) + one-shot `estIO1000` → `estIO500` SharedPrefs migration. Released + field-tested 2026-04-29..30. Two issues surfaced (both addressed in (14)).
- (14) — DSBL/TEST mapping fix + S+H multi-criteria detector code. Released + field-tested 2026-04-30.
- (15) — Launcher icon refresh (TTI bell+gear+check on green). No functional changes. **Released + field-tested 2026-05-01.**

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no Export Compliance prompt on any build.
- App icon (resolved in (15)) — `flutter_launcher_icons` wired in `pubspec.yaml`; master at `assets/icon/appicon-1024.png`. To replace in future, drop a new 1024×1024 PNG (or square JPEG) at that path and run `dart run flutter_launcher_icons`.

**Outstanding pre-Production gate (still not done):**
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at every `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission**. Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.

## 0b. Backlog — not bound to (16)

- [ ] **Messages lost while app is suspended (iOS standby)** — when the iPhone enters standby or the app is backgrounded, iOS suspends the process and the MQTT WebSocket connection drops; any alarms published during that window never reach the app on resume. Critical for a life-safety product. Likely path: route alarms through **APNs push notifications** (AWS IoT Rule → SNS → APNs) so the OS wakes the app/shows the alert independently of the in-app MQTT session. In-app MQTT then resyncs on foreground. Decide whether to keep MQTT for live foreground use only, or also add a "missed events since X" replay (retained MQTT or a tiny REST endpoint backed by IoT analytics / DynamoDB). Cross-repo: `tti-helper-mobile` + `tti-helper-aws`.

## 1. Address risks flagged in the mobile review

In priority order:

1. **OTA feedback loop** — user gets no confirmation the device accepted or completed the update. Consider emitting an OTA status/progress event on the status topic.
2. **Firmware URL validation** — no check that the manifest URL is reachable or well-formed before sending OTA command.
3. **Status cache invalidation** — `MqttService._latestStatusByThingId` is never cleared; goes stale if a device's firmware changes out-of-band. **Decision parked:** choose between Option 1 (retained MQTT message — 1-line firmware change, no mobile/AWS changes) and Option 3 (AWS IoT Device Shadow — cross-repo lift but better long-term fit for Phase 3 portal). Option 2 (mobile-triggered status republish command) deprioritized.
4. **BLE password-write try/catch** — currently swallows all exceptions. Acceptable given the firmware reboot race, but worth distinguishing expected disconnect from other failures.
5. **Wildcard status subscription scope** — `+/status` subscribes to all devices; fine for single-device users, may need tightening later.

## 2. Phase 3: lift the Device ID gating

The Welcome screen currently blocks Troubleshooting and Inspection until a Device ID is saved (via `configuredDeviceIdProvider`), and shows an amber banner prompting configuration. This is a Phase 2 workaround because the tech enters the Thing ID manually. When Phase 3 wires device binding through the TTI portal, remove the gating in `welcome_screen.dart` (banner + tile SnackBar checks) and the `configuredDeviceIdProvider` if nothing else needs it.

**Repo:** `tti-helper-mobile`
