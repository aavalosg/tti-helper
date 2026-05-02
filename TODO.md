# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(19) — migration type-cast hotfix on top of (16) parser rebuild, uploaded 2026-05-01, awaiting bench-test on Notifier panels

**Local state as of 2026-05-01:**
- `pubspec.yaml` is `1.0.0+19`. HEAD on `tti-helper-mobile/main` is `08a5fe4`, all commits pushed to origin.
- (19) IPA built 2026-05-01 21:19 at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa` (~43.6 MB). Uploaded to App Store Connect via TestFlight the same day.
- (15) released and field-tested 2026-05-01 (icon-only swap; superseded visually by (16) brand-mark icon swap, then carried forward in (19)).
- **(16) field-test blocked** by a SharedPreferences migration type-cast bug that hung the FACP picker spinner. (17) and (18) were diagnostic iterations built locally but never uploaded — both fixes (FacpModel.fromStoredName tolerant deserializer in (17), defensive try/catch wrappers in (18)) are still in (19) as forward-compat guards. (19) lands the actual fix: `prefs.get(key) is String` type-check before reading values in both `migrateLegacyEstIORename` and `migrateLegacyNotifierRename`. Verified on iPhone 16e simulator before IPA build.

**(16) ships two changes bundled:**
1. **Notifier parser manual-driven rewrite** — replaces the bench-calibrated `notify360_parser.dart` (single 2026-04-03 calibration session) with a banner-driven implementation per `tti-helper-mobile/docs/notifier_protocol.md`. Sourced from the canonical NFS-320-E (doc 52747) and NFS2-640-E (doc 52743) operations manuals. Format-identity finding: NFS-320 / NFS-640 / NFS2-640 share an identical event format; one `NotifierParser` covers all three families. New coverage over the bench-calibrated body: Pre-Alarm (`PREALM` → active), Disabled Points (`DISABL` → trouble), 640-only Mass Notification banners (`ALARM: ECS/MN`, `ACTIVE: ECS/MN`, `TROUBL: ECS/MN`), CO alarm (`ALARM: CO`), Active Fire Control, multi-line printer trouble form (`MODULE ADDRESS` keyword), `facpTime` extraction (HH:MMa MMDDYY → DateTime), and address-based `incidentKey` (`<L>M<NNN>` / `<L>D<NNN>` / `B<NN>`) with body-key fallback for address-less system-trouble lines. UI bucketing convention preserved: `ACTIVE`-prefix events still classify as `FacpEventType.active` (no scattering across security/supervisory).
2. **Launcher icon swap** — replaces the (15) bell+gear+checkmark placeholder with the official TTI brand mark (white shield with TTI lettering + flame curl, on green field). Sourced from `TTI-Icons/TTI-Icon2.jpeg` via the same recipe as (15).

**`FacpModel` rename (breaking-but-migrated):** `notify360` → three values (`notifierNfs320`, `notifierNfs640`, `notifierNfs2640`). Existing installs migrate silently to `notifierNfs2640` (B-default-modern) on first launch via `migrateLegacyNotifierRename` in `device_facp_config.dart`; users can switch in Device Management. Marker key `facp_model_migration_v16_notifier_rename`. Mirrors the (13) `estIO1000` → `estIO500` pattern.

**Test posture:** parser tests grew from 15 → 38 (23 new manual-cited fixtures); full mobile suite 176 → 199 passing. `flutter analyze` clean.

**Commits landed for (16)+(17)+(18)+(19) (all on `tti-helper-mobile/main`, pushed):**
- `bb845f1` `docs(facp): add Notifier RS-232 protocol reference (NFS-320 / NFS-640 / NFS2-640)`
- `a72c705` `docs(facp): lock CLR ACT/TB concatenated form as confirmed bench behavior`
- `c4c5e6f` `refactor(facp): split notify360 into notifierNfs320/640/2640 (steps 2/4/5 + rename half of 3)`
- `6d623e4` `refactor(facp): rewrite NotifierParser body manual-driven (step 3b)`
- `699ed60` `test(facp): expand NotifierParser coverage to manual fixtures (step 6)`
- `38b13ba` `chore(icons): switch launcher icon to TTI shield+flame brand mark`
- `7d4458e` `chore: bump build to 1.0.0+16`
- `70bebe5` `fix(facp): tolerant FacpModel deserializer; bump to 1.0.0+17` (local IPA, not uploaded)
- `723d72c` `fix(ui): defensive try/catch in save flows; bump to 1.0.0+18` (local IPA, not uploaded)
- `08a5fe4` `fix(facp): type-check before getString in migrations; bump to 1.0.0+19` (uploaded)

**Build history on App Store Connect (recent):**
- (10) — History-page event ordering under panel burst; firmware v1.0.7 OTA companion (UART 256 B → 4 KB drain-per-tick). Field-tested.
- (11) — Hybrid EST iO pairing (wrong direction — superseded by (12)).
- (12) — EST iO pairing direction corrected; `descriptionFirst => false` for all models. Field-tested 2026-04-29.
- (13) — Bundled: `_classify()` Battery Missing/Power Loss/etc descriptor pairing + full EST iO parser rewrite (17 STATUS codes, 4 address formats, iO64/iO500 + iO64/iO1000 split) + one-shot `estIO1000` → `estIO500` SharedPrefs migration. Released + field-tested 2026-04-29..30.
- (14) — DSBL/TEST mapping fix + S+H multi-criteria detector code. Released + field-tested 2026-04-30.
- (15) — Launcher icon refresh (bell+gear+check on green). Released + field-tested 2026-05-01.
- (16) — Notifier parser manual-driven rebuild + TTI shield+flame brand-mark icon. Uploaded 2026-05-01 18:34; bench-test BLOCKED by FACP-picker spinner hang (SharedPrefs migration type-cast bug). Superseded by (19).
- (17) — local IPA only, not uploaded. FacpModel.fromStoredName tolerant deserializer (forward-compat guard for stale event-DB rows). Carried forward into (19).
- (18) — local IPA only, not uploaded. Defensive try/catch wrappers around all save flows; surfaced the real root cause "type 'bool' is not a subtype of 'String?' in type cast" via SnackBar. Carried forward into (19).
- (19) — Migration type-cast fix: `prefs.get(key) is String` type-check before reading. Plus the (17) and (18) forward-compat guards. **Uploaded 2026-05-01; awaiting bench-test on Notifier panels.**

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no Export Compliance prompt on any build.
- App icon master swap recipe: drop a new 1024×1024 PNG (or square JPEG) at `assets/icon/appicon-1024.png` and run `dart run flutter_launcher_icons`.

**Outstanding pre-Production gate (still not done):**
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at every `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission**. Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.

**Resumption checklist for (19):**
- [x] All 7 (16) implementation steps merged + (17) byName guard + (18) try/catch wrappers + (19) migration type-check fix.
- [x] `flutter clean` + `flutter build ipa` 2026-05-01 21:19 (fresh from-scratch build to bust any stale cache).
- [x] Verified on iPhone 16e simulator: enter Device ID, pick Notifier model, save — saves in milliseconds with success SnackBar, no spinner hang.
- [x] Upload via TestFlight 2026-05-01.
- [ ] Install (19) from TestFlight and confirm new shield+flame icon on home screen, app switcher, Spotlight, Settings.
- [ ] Verify Save Device ID + Save FACP Model both complete in under a second on the physical device — this is the path that hung pre-(19).
- [ ] Bench-verify the parser on Notifier panels: trigger fire alarm, trouble, supervisory, security, pre-alarm, disabled point, CO (if available), system reset, system normal — confirm correct Active/History display and CLR pairing on each.
- [ ] Bench-verify on NFS-320 if accessible; NFS-640 if accessible; NFS2-640 (current bench panel).
- [ ] Confirm legacy `notify360` device entries silently migrate to `notifierNfs2640` after first launch (alongside the EstIO marker now coexisting cleanly).
- [ ] If any divergence from manual-driven behavior surfaces, capture the raw line (CloudWatch or History row) — fixtures go into `notifier_parser_test.dart` for (20).
- [ ] If a "Save failed: ..." SnackBar ever appears, screenshot the exact text — the (18) try/catch surfaces the underlying error for diagnosis.

**"What to Test" notes for the (19) External submission:**

Build 1.0.0 (19) fixes the FACP picker hang reported in build (16). When upgrading from a previous build, the FACP model picker would spin forever and never save, blocking all bench testing of the new Notifier parser. This was caused by a type-cast bug in the SharedPreferences migration that ran on first launch — fixed in this build.

After installing (19), please:

(1) Confirm the launcher icon shows the TTI shield+flame brand mark (white shield with TTI lettering and flame curl, on green background) on the home screen, app switcher, Spotlight, and Settings.

(2) Open Device Management. Enter a Device ID and tap Save Device ID — the button must NOT spin forever; you should see "Device ID saved" within a second.

(3) Pick a Notifier model (NFS-320 / NFS-640 / NFS2-640) and tap Save FACP Model — should save in under a second with "FACP model saved" confirmation. Try switching between the three Notifier options to confirm each saves cleanly.

(4) Resume the (16) Notifier parser bench testing that was blocked: trigger fire alarms, troubles, supervisory events, security events, pre-alarm warnings, and disabled points on a Notifier panel, and confirm each event displays correctly in the Active and History views. Confirm clears match their original events (CLR TB → matching TROUBLE, CLR ACT → matching ACTIVE), SYSTEM RESET silences alarms while leaving troubles in place, and SYSTEM NORMAL clears everything.

If anything fails to save, you'll now see a "Save failed: <error>" message in a SnackBar at the bottom of the screen — please screenshot and send that exact text rather than just "the spinner is stuck."

Non-Notifier panels (Fire-Lite, Vigilant, EST iO) are unchanged from build (15) and do not need re-testing.

## 0b. Backlog — not bound to (19)

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
