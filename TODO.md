# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(23) — Notifier NFS-3030 family parser, IPA building 2026-05-02, ready for upload

**Local state as of 2026-05-02:**
- `pubspec.yaml` is `1.0.0+23`. (23) IPA built 2026-05-02 (~42.5 MB) at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa`. Ready to upload to TestFlight via Transporter.
- (22) shipped as code+commit only — `flutter build ipa` ran but the IPA was NOT uploaded to TestFlight per user decision; (22)'s SYSTEM NORMAL counter fix gets implicit verification as part of (23) bench-test (which it received and passed).

**(23) implementation — new `NotifierNfs3030Parser` for the 3030 wire format:**
- Two new `FacpModel` enum values: `notifierNfs3030` (legacy 2003 panel) and `notifierNfs2_3030` (newer NFS2-3030/E). Both use the same parser. Picker grows from 8 → 10 entries.
- New `lib/core/facp/parsers/notifier_nfs3030_parser.dart` (~380 lines). Handles:
  - Banner verbs: bare/`ACKNOWLEDGED`/`ACKED`/`CLEARED` (class-specific — `ACKNOWLEDGED` for alarm-class, `ACKED` for trouble/supervisory).
  - Banner classes: FIRE ALARM, SECURITY ALARM, TROUBLE (with detail field), SUPERVISORY, plus defensive support for CO ALARM, MN ALARM, PREALARM, CO PREALARM not yet bench-observed.
  - Line-2 forms: B1 (`Zone <label> <code> <type_code> [L]? <time> <date> <addr>`), B2 (blank pad + timestamp), B3 (operator + timestamp).
  - Single-line system commands: SYSTEM RESET, SYSTEM NORMAL, TTI connected.
  - Inline `L` latching marker — no manual-cited table (huge simplification vs (21)).
  - Year-2015 panel-clock guard: `facpTime` returns null when extracted year < 2020.
  - ACK verbs route to `FacpEventType.system` so they bypass the active map and land in History only — same convention as `NotifierParser` for the 320/640 family.
- `_classify` in `alarm_provider.dart` extended to recognize 3030 banner words so banners route into the seqnbr-paired buffer (same EST iO path).
- `isNotifierFamily` extended to include both 3030 enums — gets the SYSTEM NORMAL placeholder behavior + the (22) badge=0 counter fix automatically.
- 30 new tests in `test/facp/notifier_nfs3030_parser_test.dart`. Total: 252/252 mobile tests pass (was 222 pre-(23)). `flutter analyze` clean of new issues.

**(23) bench-verified 2026-05-02:** Active+ACK+CLEAR lifecycles for SUPERVISORY/FIRE ALARM/TROUBLE; SYSTEM RESET cascade (RESET → per-event CLEAREDs → SYSTEM NORMAL); 8-event burst with ACK followups; SYSTEM NORMAL placeholder visible with badge=0; ACK events confirmed History-only; address fallback via line-2 zone descriptor confirmed for events with custom point labels (no `Detector L01D###` prefix in the banner).

**Known cosmetic gap (not blocking (23)):** A fourth line-2 form was observed during testing — `<spaces><type_code> <time> <date> <address>` (no `Zone Z001` prefix), e.g. `POWER MONITR 04:11:03P SAT MAY 02, 2026     L01M155`. The address still resolves correctly via the banner target's `_reAddress` fallback, so events land in Active fine. Only loss is that `POWER MONITR` doesn't appear in the description annotation. Add B4 form support to a follow-up build.

**"What to Test" notes for the (23) External submission:**

Build 1.0.0 (23) adds support for the Notifier NFS-3030 and NFS2-3030 fire alarm panels. Open Device Management and pick "Notifier NFS-3030" or "Notifier NFS2-3030" depending on your panel, then save.

On the panel, trigger a trouble, alarm, or supervisory event. The event should appear in the Active list with the event name and address shown, and the count badge should go up by one. Press ACK on the panel and confirm the acknowledged version goes only to History without changing the Active row or the count. Clear the condition on the panel and confirm the row leaves Active and the count drops.

Press SYSTEM RESET on the panel. The panel sends per-event clears followed by a System Normal message. The Active list should empty out and show only "SYSTEM NORMAL" with the count at zero.

Other panel families (Fire-Lite, Vigilant, EST iO, and the existing Notifier 320, 640, 2640) are unchanged from build (21).

## 0z(22). (22) closeout (historical) — SYSTEM NORMAL counter fix

(22) shipped as code+commit only on 2026-05-02 — IPA built but NOT uploaded to TestFlight per user decision; the SYSTEM NORMAL counter fix gets implicit verification as part of (23) bench-test (which it received and passed). One behavior change: synthetic SYSTEM NORMAL placeholder in the active map is now filtered out of the badge counter and bucket-filter "All"/"Others" chips via the new `isNotifierSystemNormalPlaceholder()` helper. Placeholder remains visible in the Active list as the explicit "all clear" indicator. Active-map invariant: only system-typed events ever reach `_activeMap` are these placeholders, so a `type == FacpEventType.system` check is sufficient. 222/222 tests passed pre-(23).

## 0z(21). (21) closeout (historical) — type-code-aware Notifier RESET wipe

(21) bench-tested 2026-05-02. SECOND SHOT confirmed wiped by RESET (no change). SYSTEM NORMAL counter regression → fixed in (22). iO1000 stuck trouble → deferred to a later build. (21) replaced (20)'s bucket filter with a per-event latching attribute populated by the Notifier parser:

- New `bool? latching` field on `FacpEvent`.
- `NotifierParser` stamps the attribute via `_classifyLatching` from a manual-cited type-code lookup table (~70 entries) covering Tables 3.1 / 3.2 / 3.3 / 3.4 / 3.6, with bench/engineering overrides:
  - **EVACUATE SW + SECOND SHOT** → latching (manual: N — bench-confirmed 2026-05-01)
  - **HOLD UP** → latching (engineering team — not in §3 type-code tables)
- Both dotted (`SUP.L`) and dotless (`SUPL`) detector code spellings accepted (manual is inconsistent across Table 3.4).
- Bare `CO ` (Fig 3.16) recognized as latching.
- DISABL events forced to `latching = false` (RESET doesn't clear admin-disabled points; panel re-emits while still disabled).
- system / troubleRestore events get `latching = null` (irrelevant).
- alarm_provider Notifier RESET branch becomes `_activeMap.removeWhere((_, v) => v.latching == true)`.

Per-parser documentation also added in `tti-helper-mobile/docs/`:
- `parsers_README.md` (index + cross-cutting state-machine notes)
- `notifier_parser.md` (testing-oriented guide, complements existing `notifier_protocol.md`)
- `firelite_9050_parser.md`, `estio_parser.md`, `vigilant_vm1_parser.md` (first-cut for those families).

220/220 mobile tests pass (parser tests grew to 67 with 16 new latching cases; alarm_active_map tests grew to 17 with 5 new RESET-by-latching cases).

## 0z. (19) closeout (historical) — Notifier parser manual-driven rebuild + brand-mark icon, uploaded 2026-05-01, FIELD-TESTED OK

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
