# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(16) — Notifier parser rebuild + brand-mark icon, uploaded 2026-05-01, awaiting bench-test on Notifier panels

**Local state as of 2026-05-01:**
- `pubspec.yaml` is `1.0.0+16`. HEAD on `tti-helper-mobile/main` is `7d4458e`, all commits pushed to origin.
- (16) IPA built 2026-05-01 18:32 at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa` (~43.6 MB). Uploaded to App Store Connect via Transporter the same day; processing started 18:34.
- (15) released and field-tested 2026-05-01 (icon-only swap; superseded visually by (16)).

**(16) ships two changes bundled:**
1. **Notifier parser manual-driven rewrite** — replaces the bench-calibrated `notify360_parser.dart` (single 2026-04-03 calibration session) with a banner-driven implementation per `tti-helper-mobile/docs/notifier_protocol.md`. Sourced from the canonical NFS-320-E (doc 52747) and NFS2-640-E (doc 52743) operations manuals. Format-identity finding: NFS-320 / NFS-640 / NFS2-640 share an identical event format; one `NotifierParser` covers all three families. New coverage over the bench-calibrated body: Pre-Alarm (`PREALM` → active), Disabled Points (`DISABL` → trouble), 640-only Mass Notification banners (`ALARM: ECS/MN`, `ACTIVE: ECS/MN`, `TROUBL: ECS/MN`), CO alarm (`ALARM: CO`), Active Fire Control, multi-line printer trouble form (`MODULE ADDRESS` keyword), `facpTime` extraction (HH:MMa MMDDYY → DateTime), and address-based `incidentKey` (`<L>M<NNN>` / `<L>D<NNN>` / `B<NN>`) with body-key fallback for address-less system-trouble lines. UI bucketing convention preserved: `ACTIVE`-prefix events still classify as `FacpEventType.active` (no scattering across security/supervisory).
2. **Launcher icon swap** — replaces the (15) bell+gear+checkmark placeholder with the official TTI brand mark (white shield with TTI lettering + flame curl, on green field). Sourced from `TTI-Icons/TTI-Icon2.jpeg` via the same recipe as (15).

**`FacpModel` rename (breaking-but-migrated):** `notify360` → three values (`notifierNfs320`, `notifierNfs640`, `notifierNfs2640`). Existing installs migrate silently to `notifierNfs2640` (B-default-modern) on first launch via `migrateLegacyNotifierRename` in `device_facp_config.dart`; users can switch in Device Management. Marker key `facp_model_migration_v16_notifier_rename`. Mirrors the (13) `estIO1000` → `estIO500` pattern.

**Test posture:** parser tests grew from 15 → 38 (23 new manual-cited fixtures); full mobile suite 176 → 199 passing. `flutter analyze` clean.

**Commits landed for (16) (all on `tti-helper-mobile/main`, pushed):**
- `bb845f1` `docs(facp): add Notifier RS-232 protocol reference (NFS-320 / NFS-640 / NFS2-640)`
- `a72c705` `docs(facp): lock CLR ACT/TB concatenated form as confirmed bench behavior`
- `c4c5e6f` `refactor(facp): split notify360 into notifierNfs320/640/2640 (steps 2/4/5 + rename half of 3)`
- `6d623e4` `refactor(facp): rewrite NotifierParser body manual-driven (step 3b)`
- `699ed60` `test(facp): expand NotifierParser coverage to manual fixtures (step 6)`
- `38b13ba` `chore(icons): switch launcher icon to TTI shield+flame brand mark`
- `7d4458e` `chore: bump build to 1.0.0+16`

**Build history on App Store Connect (recent):**
- (10) — History-page event ordering under panel burst; firmware v1.0.7 OTA companion (UART 256 B → 4 KB drain-per-tick). Field-tested.
- (11) — Hybrid EST iO pairing (wrong direction — superseded by (12)).
- (12) — EST iO pairing direction corrected; `descriptionFirst => false` for all models. Field-tested 2026-04-29.
- (13) — Bundled: `_classify()` Battery Missing/Power Loss/etc descriptor pairing + full EST iO parser rewrite (17 STATUS codes, 4 address formats, iO64/iO500 + iO64/iO1000 split) + one-shot `estIO1000` → `estIO500` SharedPrefs migration. Released + field-tested 2026-04-29..30.
- (14) — DSBL/TEST mapping fix + S+H multi-criteria detector code. Released + field-tested 2026-04-30.
- (15) — Launcher icon refresh (bell+gear+check on green). Released + field-tested 2026-05-01.
- (16) — Notifier parser manual-driven rebuild + TTI shield+flame brand-mark icon. **Uploaded 2026-05-01 18:34; awaiting bench-test on Notifier panels.**

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no Export Compliance prompt on any build.
- App icon master swap recipe: drop a new 1024×1024 PNG (or square JPEG) at `assets/icon/appicon-1024.png` and run `dart run flutter_launcher_icons`.

**Outstanding pre-Production gate (still not done):**
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at every `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission**. Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.

**Resumption checklist for (16):**
- [x] All 7 implementation steps merged (protocol doc + parser body rewrite + tests + icon swap + pubspec bump).
- [x] `flutter build ipa` 2026-05-01 18:32.
- [x] Upload via Transporter 2026-05-01 18:34. Transporter sidebar shows the (15) icon next to the (16) entry — known cache behavior; the embedded IPA contains the new icon and App Store Connect will show it once processing finishes.
- [ ] Install (16) from sideload (or TestFlight once processed) and confirm new shield+flame icon on home screen, app switcher, Spotlight, Settings.
- [ ] Bench-verify the parser on Notifier panels: trigger fire alarm, trouble, supervisory, security, pre-alarm, disabled point, CO (if available), system reset, system normal — confirm correct Active/History display and CLR pairing on each.
- [ ] Bench-verify on NFS-320 if accessible; NFS-640 if accessible; NFS2-640 (current bench panel).
- [ ] Confirm legacy `notify360` device entries silently migrate to `notifierNfs2640` after first launch.
- [ ] If any divergence from manual-driven behavior surfaces, capture the raw line (CloudWatch or History row) — fixtures go into `notifier_parser_test.dart` for (17).

**"What to Test" notes for the (16) External submission:**

Build 1.0.0 (16) ships two changes:

(1) The launcher icon is replaced with the official TTI brand mark — a white shield with TTI lettering and a flame curl on a green background. After installing, please confirm the new icon shows up on your home screen, in the app switcher, in Spotlight search, and in Settings under TTI Helper.

(2) The parser for Notifier fire alarm panels (NFS-320 / NFS-640 / NFS2-640 families) has been rewritten to follow the official Notifier operations manuals. If you are testing on a Notifier panel, please trigger fire alarms, troubles, supervisory events, security events, pre-alarm warnings, and disabled points, and confirm each event displays correctly in the Active and History views. Also confirm that clears match their original events (a CLR TB clearing its TROUBLE, a CLR ACT clearing its ACTIVE), and that a SYSTEM RESET silences alarms while leaving troubles in place, while SYSTEM NORMAL clears everything. The panel model picker in Device Management now shows three Notifier options (NFS-320, NFS-640, NFS2-640) instead of one combined option — your existing selection automatically migrates to NFS2-640; switch in Device Management if you have an NFS-320 or NFS-640 panel.

Non-Notifier panels (Fire-Lite, Vigilant, EST iO) are unchanged from build (15) and do not need re-testing.

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
