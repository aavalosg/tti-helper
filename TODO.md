# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(27) — Inspection feature: iO walk-test + normal-mode capture, two-line pairing reuse, PDF observation/raw-line surfacing. IPA built + uploaded 2026-05-04, awaiting field-test.

**Local state as of 2026-05-04:**
- `pubspec.yaml` is `1.0.0+27`. Mobile HEAD `3bae3c9`, parent HEAD `b47fc89` — both pushed.
- (27) IPA built 2026-05-04 22:42:55 (~42.5 MB) at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa` and uploaded to TestFlight via Transporter the same evening. Bench-verified on iO500 panel during the build session; awaiting physical-device field-test on iPhone via TestFlight install (scheduled 2026-05-05).
- (26) field-tested OK 2026-05-04 (Fire-Lite 9600 alarm-mode lifecycle on physical panel; walk-test mode revealed gaps that triggered the (27) work).
- 298/298 mobile tests pass. `flutter analyze` clean of new issues.

**"What to Test" notes for the (27) External submission:**

Build 1.0.0 (27) is a major upgrade to the Inspection feature for EST iO panels. Pre-(27), the Inspection feature only captured iO walk-test events (TEST ACT) into Testing Areas — normal-mode events like Pull Station alarms, smoke detector alarms, and supervisory activations silently dropped without ever reaching the inspection record.

After installing (27), please:

(1) Open Inspection from the home screen, start a new session for an iO panel, mark a zone Active. Trigger a Pull Station, a Smoke Detector, and a Supervisory device (e.g. a tamper switch or PIV) on the panel. Each event should appear as a row in the Testing Areas tab with the device's programmed custom label as the description (or "Loop X Device Y" / "NAC NN" if no custom label is set), the FACP zone in the location field, and the panel's authoritative header line (e.g. `PULL ACT | … L:1 D:187`) shown beneath in monospace.

(2) Press SYSTEM RESET on the panel. The inspection rows should stay — only the troubleshooting Active list clears. You can then mark each row Pass / Fail / N/A / Pending, optionally adding a per-event observation in the row's NOTES field. At the end of the zone, type an area-level "Notes" / observation in the bottom textfield — that becomes the "General Observation" block at the foot of the zone in the printed report.

(3) Switch the panel into Walk Test mode and trigger a couple of devices. Each TEST ACT event should be captured the same way as normal-mode events; TEST RST events from the panel should NOT create separate rows (the active map clears them silently). Walk Test entry/exit signals (TRBL ACT/RST E:047) are intentionally not captured — they're panel-internal mode changes, not device tests.

(4) Print the inspection report. N/A entries should NOT appear in the per-zone tables. Each zone with a typed area-level note should show a labeled "General Observation" block at its foot, with a dark left accent and grey background. The DESCRIPTION column on each event row should read `<device label> <raw FACP header>` (e.g. `1St flgptwdkmtmpgmadPIV SUPV ACT | … L:1 D:126`).

Other panel families: Fire-Lite walk-test parsing was also fixed in this build — if you have a Fire-Lite 9050 or 9600 available, please trigger devices in walk-test mode and confirm events land in Testing Areas with proper ADDRESS columns (no empty cells, no raw timestamps in the description). Notifier walk-test inspection is not yet addressed — that's deferred to (28). Troubleshooting (Live Events) on all families is unchanged from (26) except for one bucket-routing change: iO supervisory events (SUPV ACT) now appear under "Others" instead of "Trouble".

**(27) iO scope shipped (this session, bench-verified on iO500):**

1. **Inspection filter widened for iO 4-letter ACT verbs** (`zone_event_provider.dart:_isTestingAreaEvent`). Pre-(27) only Fire-Lite/Notifier verb forms (`ALARM`/`PRE-ALARM`/`ACTIVE`/`TEST`) matched the prefix gate, so iO normal-mode events (`PULL ACT`, `ALRM ACT`, `SUPV ACT`, etc.) silently dropped — Inspection only ever worked for iO inside walk-test mode where `TEST ACT` happened to match. Added 12 explicit iO `<MNEM> ACT` prefixes (Table 41 alarm-class + supervisory). RST suffix verbs intentionally excluded (the active-map clear via incidentKey still fires; just no separate Testing Areas row).
2. **TEST narrowed to TEST ACT only** — drops the panel's `TEST RST` rows from Testing Areas. Per-event-row inspection record stays clean; the troubleshooting active map still pairs ACT/RST via incidentKey.
3. **iO `SUPV` / `COSU` re-routed: `FacpEventType.trouble` → `FacpEventType.supervisory`** (`estio_parser.dart:_eventType`). Reverts the 2026-04-29 decision because type-aware downstream logic (inspection layer) now treats supervisory as a real device-condition class. Bucket on Live Events moves Trouble → Others. `_updateActiveMap` already handles supervisory identically to alarm/active so no clearing-behavior change. Existing parser tests updated: `SUPV ACT` and `COSU ACT` now expect `supervisory` + `EventBucket.others`.
4. **Architectural fix: Inspection consumes the parsed-event stream from `alarm_provider`** instead of subscribing to raw MQTT. New `StreamController<FacpEvent>.broadcast()` on `AlarmNotifier`, emitted in `_emitEvent` after the seqnbr-keyed line1+line2 pairing buffer matches. `ZoneEventNotifier.build()` reads `alarmProvider.notifier.parsedEvents` (using `ref.read` not `ref.watch` so state changes don't tear down the subscription). Effect: descriptor (custom device label) is correct in inspection rows because the parser now sees both lines. Eliminates the parallel-consumer flaw the inspection feature was originally bolted on with.
5. **Fire-Lite asterisk-time fix** (`firelite_parser.dart:_re`) — regex separator `\d{2}:\d{2}[AP]` → `\d{2}[:*]\d{2}[AP]`. Walk-test events on the 9600 (where the panel substitutes `*` for `:` as a printer-port walk-test marker) now parse on the same path as alarm-mode and route to alarm/supervisory buckets normally instead of falling through to the system-event branch with empty zone. 5 new test fixtures including a regression guard for the `:` time form. Bench-dump for Fire-Lite walk-test still pending (the regex fix is grounded in Image 1/2 PDF screenshot evidence, not on a captured wire dump).
6. **`ZoneEventEntry.rawLine1` field** + **DB migration v3 → v4** (`inspection_database.dart`): new `raw_line1 TEXT` column on `zone_event_entries`, populated from `event.rawLine1` in `_onEvent`. Surfaced on the Testing Areas card (in monospace below time/zone) so the inspector sees the same authoritative header line that Live Events shows.
7. **Inspection PDF**:
    - **N/A entries dropped from per-zone tables** (per-row N/A no longer prints to AHJ). Zone-level filter intentionally NOT applied — zones with N/A result still render so their area-level note ("General Observation") prints. Summary line: "Pass: X  Fail: Y  Pending: Z" (N/A counter removed).
    - **"General Observation" block** at the foot of every zone — labeled, bordered, dark left accent, 9pt body. Replaces the old 8pt grey one-liner that was easy to miss. Always renders if `zone.notes` is non-empty, regardless of zone result.
    - **DESCRIPTION column shows `description + ' ' + rawLine1`** — parsed device label first (e.g. `1St flgptwdkmtmpgmadPIV`), then a single space, then the panel's authoritative header line (e.g. `SUPV ACT | … L:1 D:126`). Falls back to just `description` for legacy rows without `rawLine1`.

**(27) iO bench-verification on iO500 panel 2026-05-04 (in this session):**
- iO normal mode: PULL ACT / ALRM ACT / SUPV ACT all captured to Testing Areas with custom device labels and proper bucket routing.
- iO walk-test mode: TEST ACT for L:1 D:126 (PIV), L:1 D:187 (smoke), E:130 / E:131 (NAC 03/04) all captured. TRBL ACT/RST E:047 walk-test entry/exit signals correctly skipped (panel-internal, not device tests).
- TEST RST drops from inspection layer; troubleshooting Active list still clears the corresponding TEST ACT entry.
- General Observation block + DESCRIPTION column rendering verified on PDF print.

**(27) still pending before build/upload:**
1. **Fire-Lite walk-test bench dump** — disconnect iO, connect 9600, capture a walk-test session into `facp_manual/Firelite/bench_9600_raw/raw_<date>_walktest.txt` to verify the asterisk-time hypothesis against real wire bytes (currently grounded in PDF-screenshot evidence only). Apply the analogous filter widening if needed (Fire-Lite walk-test should already capture via existing `ALARM:` / `ACTIVE` / `PRE-ALARM` prefixes since the verbs don't change in walk-test mode).
2. **Notifier walk-test bench dump** — same exercise on a Notifier panel. Wire form completely unverified for walk-test; might need a similar filter widening or a different mechanism entirely.
3. **Description column refactor (parked, post-(27))** — see § 0b. The current `description + ' ' + rawLine1` mash works but isn't the long-term shape; needs a structured cell with proper hierarchy.

**(26) "What to Test" notes** — kept as historical reference below.

## 0z(26). (26) closeout (historical) — Fire-Lite MS-9600LS family + shared FireLiteParser, FIELD-TESTED OK 2026-05-04

(26) shipped + field-tested 2026-05-04 in alarm mode on a physical MS-9600LS panel. Full TROUBL/CLEARt/ACTIVE/CLEARe/ALARM:/CLEARa lifecycle worked; OFF NORMAL summary placeholder filtering works as designed. Walk-test mode revealed two follow-up gaps — (i) Fire-Lite asterisk-time format `HH*MMA/P` failing the parser regex, (ii) inspection layer's prefix filter not recognizing iO 4-letter mnemonics — both addressed in (27). Bench-evidence: 31 events captured 2026-05-02 on real MS-9600LS panel — saved to `facp_manual/Firelite/bench_9600_raw/raw_20260502_195435_build25.txt`. 9050 + 9600 share an identical single-line printer/RS-232 wire format — confirmed against MS-9600LS Series Manual P/N 52646:B8 §4.4-4.6 + Table 3.1 (page 79). 50 fixtures total in `firelite_parser_test.dart`, 293/293 mobile tests passed pre-(27). Implementation details + commit list preserved at the prior (26) section now retained below for historical reference.

**(26) historical local state (frozen 2026-05-02):**
- `pubspec.yaml` was `1.0.0+26`. HEAD `2f2d1a3` (mobile) — pushed. (26) IPA built 2026-05-02 21:15 (~43.6 MB) at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa` and uploaded to TestFlight via Transporter the same evening. The simulator-only "connection unit test" passed; full lifecycle bench-verify deferred to the physical-FACP test on a real MS-9600LS panel via TestFlight install. **2026-05-04: bench-verify done — passed.**
- (25) uploaded 2026-05-02 18:03 — AppBar refresh fix.
- (24) uploaded 2026-05-02 17:40 then **ABANDONED** (AppBar-stale bug + Apple version-collision blocking re-upload at v24).
- (23) uploaded 2026-05-02 16:16 — Notifier 3030 parser. Field-tested OK 2026-05-02; ready to release.

**(26) implementation — Fire-Lite MS-9600LS family parser + shared `FireLiteParser`:**
- One new `FacpModel` enum value: `firelite9600` (covers MS-9600LS / LSE / UDLS / UDLSE / LSC — all SKUs share the same wire format per the manual). Picker grows from 10 → 11 entries; newest-first order puts 9600 above 9050.
- `lib/core/facp/parsers/firelite_9050_parser.dart` → `firelite_parser.dart`. Class `FireLite9050Parser` → `FireLiteParser`, constructor takes `FacpModel`. Same pattern as Notifier 320/640/2640 sharing `NotifierParser` and EST iO500/iO1000 sharing `EstIOParser`.
- Bench-evidence basis: 31 events captured 2026-05-02 on a real MS-9600LS panel — full TROUBL/CLEARt/ACTIVE/CLEARe/ALARM:/CLEARa lifecycle, RESET cascade, OFF NORMAL pair, 4-NAC silencing. Saved to `facp_manual/Firelite/bench_9600_raw/raw_20260502_195435_build25.txt`. Verdict: 9050 + 9600 share an identical single-line printer/RS-232 wire format — confirmed against MS-9600LS Series Manual P/N 52646:B8 §4.4-4.6 + Table 3.1 (page 79).
- New UI filter: `isFireLiteOffNormalPlaceholder()` in `alarm_provider.dart`. The 9600 emits `TROUBL IN SYSTEM OFF NORMAL MESSAGE` whenever any condition is active and `CLEARt` of same when normal — the panel's implicit "all clear" indicator. Filter excludes it from the badge count and bucket-filter chips so the user doesn't see N+1 troubles when N other faults exist; row stays visible in Active list. Mirrors the (22) Notifier SYSTEM NORMAL placeholder UX pattern.
- `_reSupervBody` extended for MEDIC-ALERT / HAZARD-ALERT / TORNADO-ALERT (manual-cited supervisory-latching codes from Table 3.1).
- Per user direction: PRE-ALARM and DSBL/disabled-point handling NOT implemented in (26) (deferred). PULL STATION uses the same path as 9050 (verb-routed via `ALARM:`).
- Latching attribute stays null for Fire-Lite (RESET is a panel-spec no-op; panel-driven cascade does the clearing). Bench-reconfirmed 2026-05-02 on the 9600.
- New parser doc: `docs/firelite_parser.md` (replaces `firelite_9050_parser.md` + `firelite_9050_9600_plan.md`). Manual-cited spec covering both panels + evidence-vs-inference table for 9600 features (MNS EVENT, HVAC OVRRIDE/RESTART listed as inferred-only — no printer-output sample in the manual, awaiting a panel with these programmed).
- Tests: 50 fixtures total in `firelite_parser_test.dart` (15 ported 9050 + 30 new 9600 bench fixtures + 4 OFF NORMAL filter + 1 multi-loop address). `flutter analyze` clean of new issues. 293/293 mobile tests pass (was 270/270 pre-(26)).

**"What to Test" notes for the (26) External submission:**

Build 1.0.0 (26) adds support for the Fire-Lite MS-9600LS family panels (MS-9600LS / LSE / UDLS / UDLSE / LSC). Open Device Management, tap CHANGE on the FACP Model row, and pick **Fire-Lite MS-9600LS** under the Fire-Lite brand. The app bar should immediately show "MS-9600LS" in green.

On the panel, trigger any combination of troubles, alarms, and supervisory events. Each one should appear in Active and the count badge should reflect the number of real conditions. Note that the panel also emits a summary "OFF NORMAL MESSAGE" whenever any condition is active — this row appears in the Active list as the panel's "all clear" indicator but is intentionally excluded from the count badge so you don't see N+1 troubles when N actual faults exist.

Press ACK on the panel — Active list unchanged. Press SIGNAL SILENCE — Active list unchanged. Press SYSTEM RESET — the panel emits a per-event clear cascade followed by RESTOR; any still-physical conditions immediately re-fire as fresh troubles (this is the panel's normal behavior). Clear faults at the source and confirm matching CLEARt rows pair off correctly.

Other panel families (MS-9050, Notifier 320/640/2640/3030, EST iO, Vigilant) are unchanged from (25). Switch the picker back to MS-9050 mid-session and confirm the existing 9050 lifecycle still works as before.

## 0z(25). (25) closeout (historical) — AppBar refresh fix, uploaded 2026-05-02, awaiting field-test

**Local state as of 2026-05-02:**
- `pubspec.yaml` was `1.0.0+25`. (25) IPA built 2026-05-02 18:03 (~42.5 MB) and uploaded to TestFlight the same evening; awaiting field-test on the iPhone via TestFlight install. Same code as (24) plus a one-line fix: `ref.invalidate(deviceFacpConfigProvider)` in the picker save flow so the Welcome / MainShell AppBar consumers actually see the new model after a CHANGE save (the cached DeviceFacpConfig wrapper instance never changed identity, so `ref.watch` skipped the rebuild even though SharedPreferences had the new value).
- (24) was uploaded 2026-05-02 17:40 then **ABANDONED** by the user — has the AppBar-stale bug AND a re-upload of fixed code at version 24 is rejected by Apple ("bundle version must be higher"). User decided not to test (24); (25) supersedes it.
- (23) uploaded 2026-05-02 16:16 — Notifier 3030 parser. **Field-tested OK 2026-05-02** by user; ready to release.

**(24) implementation — FACP model picker UX refactor + AppBar contextualization:**
- Three new getters on `FacpModel`: `String get brand` (e.g. 'Notifier'), `String get shortName` (e.g. 'NFS2-3030'), `String get descriptionDetail` (the subtitle text moved off the Device Management + Provisioning screens). Plus top-level helpers `allBrands` (alphabetical with Unknown last) and `modelsByBrand(brand)` (newest-first within brand).
- New file `lib/core/facp/presentation/facp_picker.dart` — `FacpBrandPickerScreen` and `FacpModelPickerScreen`. Brand picker has a search field that filters across brands + model names + descriptions; tapping a brand row pushes the model picker scoped to that brand. Search results in the brand picker show models grouped by brand, tap to save directly. Defensive try/catch + SnackBar around the save flow (mirrors the (18) `_saveDevice` guard).
- Two new routes in `lib/core/router.dart`: `/facp-picker/brand` (extra: thingId) and `/facp-picker/model` (extra: `FacpModelPickerArgs`).
- `device_management_screen.dart` — replaced inline `RadioListTile` block with a "Currently configured: X / CHANGE ›" InkWell that opens the picker. Removed dead code (`_saveModel`, `_modelSubtitle`, `_savingModel`). Section-header icons dropped; `_SectionHeader` simplified to title+subtitle only. New `_openFacpPicker` extends the active-session block (already on Device ID save) to FACP model save — dialog blocks change while a session is open.
- `provisioning_screen.dart` — kept inline radio (lifecycle is different — no thingId yet), but swapped `_modelSubtitle` for `FacpModel.descriptionDetail` and dropped section-header icons.
- `main_shell.dart` AppBar — drops "TTI Helper" brand title on contextual screens. Promotes session location to primary header. Shows `FacpModel.shortName` in accent green as subtitle (or "Generic parser" in dim gray if model is unknown). Async-load guard: subtitle hidden while `deviceFacpConfigProvider` is still loading on cold start, avoids the "Generic" → real-model flash.
- `welcome_screen.dart` AppBar — keeps "TTI Helper" brand title (this is the workflow-selection home, no session yet) but adds the configured FACP model as a subtitle so the tech sees the bound panel before starting a session. Hidden when no device is configured.
- Tests: 18 new unit tests on the new getters/helpers in `facp_model_test.dart`. Total 270/270 mobile tests pass (was 252 pre-(24)). `flutter analyze` clean of new issues. Widget tests deferred — codebase has no widget-test scaffolding yet, manual simulator verification covered the picker flow.

**(24) bench-verified 2026-05-02 on the simulator** with build (24) installed. Welcome AppBar shows TTI Helper + NFS2-3030 in green. Other flows (CHANGE button → brand picker → model picker → Save → Device Management updates) tested visually.

**Risk-class check vs (16) FACP-picker spinner regression:** (24) does NOT rename any enum values — no SharedPreferences migration code, no schema change, same `setString(key, model.name)` round-trip as today. The (16) bug class can't recur. New risk surface is navigation + async loading state, both mitigated (defensive try/catch + valueOrNull guard).

**"What to Test" notes for the (25) External submission:**

Build 1.0.0 (25) is a small patch on top of (24). Same Device Management refresh and FACP model in the app bar — plus one fix: when you change the panel model from the new picker, the app bar now updates immediately to show the new model name. In (24) the app bar still showed the previous model until you closed and reopened the app.

To verify, open Device Management, tap CHANGE on the FACP Model row, pick a different brand and model, tap Save FACP Model, and confirm the app bar on Welcome (and on Active and History after starting a session) flips to the new model name on the spot.

Everything else from (24) still applies: brand-then-model picker with search, session location as the primary app bar header, the active-session block on FACP model change, and no parser changes from (23). All panel families behave the same.

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

## 0z(23). (23) closeout (historical) — Notifier NFS-3030 family parser, FIELD-TESTED OK

(23) shipped + field-tested OK 2026-05-02. New `NotifierNfs3030Parser` for the 3030 wire format dialect: distinct verbs (ACKNOWLEDGED/ACKED/CLEARED), loop-prefixed addresses, two-line emit pattern (banner + zone-or-system descriptor, paired by seqnbr ~80–180 ms apart), inline `L` latching marker on the zone descriptor. Bench-verified on a real NFS2-3030 panel during the same session — full lifecycle (active / ACK / CLEAR / SYSTEM RESET cascade / SYSTEM NORMAL placeholder with badge=0) confirmed working. ACK verbs route to `FacpEventType.system` so they land in History only. 30 new tests; 252/252 passing pre-(24).

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

## 0b. Backlog — not bound to (27)

- [ ] **Inspection PDF: refactor the DESCRIPTION column** — the (27) implementation glues `description` + `' '` + `rawLine1` into a single string for the column cell. Works as a quick-fix for AHJ-readable raw context, but the long-term shape should be a structured two-line cell (parsed description as the primary line, raw FACP header as a secondary line in monospace grey 6pt) so the visual hierarchy mirrors the Live Events / Testing Areas card layout. Currently both lines share the same 7pt style and look like one mashed string. Slot: any post-(27) build. Cross-repo: `tti-helper-mobile` only.

- [ ] **Messages lost while app is suspended (iOS standby)** — when the iPhone enters standby or the app is backgrounded, iOS suspends the process and the MQTT WebSocket connection drops; any alarms published during that window never reach the app on resume. Critical for a life-safety product. Likely path: route alarms through **APNs push notifications** (AWS IoT Rule → SNS → APNs) so the OS wakes the app/shows the alert independently of the in-app MQTT session. In-app MQTT then resyncs on foreground. Decide whether to keep MQTT for live foreground use only, or also add a "missed events since X" replay (retained MQTT or a tiny REST endpoint backed by IoT analytics / DynamoDB). Cross-repo: `tti-helper-mobile` + `tti-helper-aws`.

- [ ] **Vigilant — manual-driven parser rebuild + add VS family** — the current `VigilantVm1Parser` was calibrated from CloudWatch logs only (no manual on hand at the time). Full doc set is now in tree at `tti-helper-mobile/facp_manual/Vigilant/`:
    - **VM**: Kidde VM-1 Technical Reference (`3101890-EN-R006`, June 2018 — primary), Operating Instructions, Users Guide, K85005-0068 Local Operator Console, K85005-0134 Submittal Guide, K85005-0138 Life Safety Control System.
    - **VS**: Kidde Vigilant VS1+VS2 Installation/Operations/Programming 2010, plus the VS2-specific manual.

    **VM-1 audit findings (2026-05-03, against `3101890-EN-R006`):**

    *Critical caveat* — the TR documents what the panel **logs/displays** (event queues, Details screen, History report) but does **not publish a byte-level printer/RS-232 wire-format spec**. MIR-PRT/S section (TR pp. 95-98) is hardware-only (DB-25, DIP, 4800-8N1). HyperTerminal section (pp. 164-165) shows no sample output. So the manual *will not* eliminate the need for a bench dump — it sharpens *where to look* and exposes one outright behavior bug.

    What the manual *confirms*:
    - Two-line emit pattern (banner + descriptor) — TR p. 17, Fig 6.
    - 24-hour `HH:MM:SS MM/DD/YYYY` — TR Fig 6.
    - Verbs `ALARM ACTIVE` and `COMMON TRBL ACT` printed verbatim — TR pp. 17, 39. (Manual does NOT enumerate the rest of the verb dictionary.)
    - Restoration is per-point — TR p. 36.

    What the manual *contradicts* (likely parser bugs):
    - **🚨 Wipe-on-RESET is wrong.** TR p. 38: *"Pressing Reset restores the system to normal, provided all latched inputs have been restored."* Active alarms whose source is still asserted **survive Reset**. No documented `SYSTEM NORMAL` sweep, no per-event `CLEARED` cascade. Behavior is closer to Fire-Lite (RESET is effectively a no-op for the parser) than Notifier 3030. Today's `vigilant_vm1_parser.md:88` build-3 fall-through wipes everything — drop that for Vigilant in the rebuild.
    - **PRE-ALARM is missing from the verb list.** TR Table 16 p. 27 lists primary/secondary states as Alarm / **Prealarm** / Trouble. Parser (`vigilant_vm1_parser.dart:120-140`) routes prealarm rows to `system` (no-op) today.
    - **ACK is metadata, not a log line.** TR p. 37: ACK adds a check-mark + "Acknowledged" tag on the existing queue entry, not a separate event class. The CloudWatch-observed `ACTIVATE RESET` / `ACTIVATE ALM SILENCE` standalones are **operator-command echoes** (TR Tables 8-16, pp. 20-27), belong on the operator-command track only.
    - **Address format is concatenated decimal `PPCCDDDD`** (8 digits) per TR Appendix B p. 190 — not hex. Parser regex accepts hex (`[0-9A-Fa-f]+`); narrow to digits-only in the rebuild. The `P:nn C:nn D:nnnn` form the parser sees is the LCD Details rendering (TR Fig 10 p. 39).

    What the manual is *silent on* — bench data still required:
    - Comprehensive banner verb dictionary (`TRBL ACT`, `TAMPER ACTIVE`, `MNTR ACT`, `PULL STATION ACT`, etc. all came from CloudWatch, not the manual).
    - The `::` separator (panel/firmware quirk; not documented).
    - `-OPERATOR COMMAND-` literal log-line format.
    - Per-event latching attribute — TR p. 46 says latching is a **device-programming property**, not a banner-string attribute. We can't derive it from the verb the way Notifier 320/640 lets us; either accept the limitation or get a programming dump.

    Dialect risks the manual flags:
    - **Bilingual installs** (TR p. 26 Toggle Language) can emit primary message text in French/Spanish mixed with English — regex on English banner strings will silently drop events.
    - C-CPU firmware 1.x scope only.

    **VS1/VS2 audit findings (2026-05-03, against `3101113 Rev 4.0`, ISS 02MAR10, GE Security):**

    *Note* — unlike the VM TR, the VS manual *does* publish an explicit printer wire-format spec with worked samples (Ch.3 "Event printout examples", pp.145-146). The `Kidde-VS2-Manual.pdf` is the same document rebranded — one read covers both panels. Far more authoritative than the VM source.

    What the VS manual documents:
    - **Two-line emit, pipe-separated:** `<VERB> | HH:MM:SS MM/DD/YYYY <ADDRESS>` line 1, descriptor line 2. The separator is a literal ` | ` (space-pipe-space) — *not* the `::` quirk seen on VM-1.
    - **Closed verb dictionary** (Table 39, pp.144-145, two columns LCD-vs-Printer): `SMK ACT`, `HEAT ACT`, `DUCT ACT`, `PULL ACT`, `WFLW ACT`, `ALRM ACT`, `SUPV ACT`, `MON ACT`, `PALM ACT`, `ALMV ACT`, `MANT ACT`, `TRBL ACT`, `DSBL ACT`, `TEST ACT` — all `<4-LETTER MNEMONIC> ACT`. Latching variants of supervisory inputs print **identically** — latch attribute not encoded in the verb.
    - **Four address prefix forms:** `L:N D:NNN` (loop+device, two separate fields), `Z:NNN` (zone), `A:NNN` (annunciator), `E:NNN` (internal/panel event keyed to the Event ID dictionary). Loops 1-2, devices 001-254.
    - **Event ID dictionary** (Table 40, pp.146-150) — 102 entries mapping `E:NNN` to human-readable system/internal events: 020 startup, 022 Reset, 024 Panel silence, 025 Signal silence, 026 Drill, 027 Walk test, 029 Clear history, 034 Ground fault, 036 Battery low, 038/041 AC power, 042 Common alarm, 043 Common supv, 044 Common monitor, 056-057 Net recvr faults, 058-061 NAC1-4 trouble, 062 Printer trouble, 063-070 Annunciator 1-8 trouble, 071-102 Zone 1-32 (active/trbl/dsbl/palm/almv/mant/test).
    - **24-hour `HH:MM:SS MM/DD/YYYY`**, 4-digit year — same as VM-1.
    - **RESET semantics match VM** (p.151 verbatim parallel of VM TR p.38): active conditions survive Reset. Reset emits Event 022 only, no per-event CLEAR cascade. ACK emits Event 024/025 only — metadata-not-log-line, same as VM.
    - **VS1 vs VS2 is hardware-only** (loop count, zone capacity); wire format byte-identical. One parser class covers both.
    - **No bilingual / Toggle Language feature** — not present in this 2010 GE-Security manual (a later VM-era addition).
    - **9600-8N async** RS-232 (vs VM-1's 4800).

    What the VS manual is *silent on* — bench data still required:
    - **Restore/clear verb form.** Manual nowhere shows a `*** RST` or `*** CLR` printer string. History report references "event-state restoration" (p.168) but the wire format for a restoration line is silent. **Bench dump required** for restore handling — likely the only blocker before VS parser coding.
    - Per-event sequence/index numbers on the wire (LCD shows `001`-style queue position but printer samples don't include it).

    **VM-vs-VS verdict — distinct dialects, separate parser classes:**

    | Dimension | VM-1 | VS1/VS2 |
    |---|---|---|
    | Separator | `::` (panel quirk) | ` \| ` (pipe, manual-spec) |
    | Verbs | `ALARM ACTIVE` / `COMMON TRBL ACT` (manual sparse; rest from CloudWatch) | 14 enumerated `<MNEM> ACT` mnemonics, Table 39 |
    | Address | `PPCCDDDD` 8-digit decimal concatenated (LCD: `P:nn C:nn D:nnnn`) | 4 typed prefix forms: `L:N D:NNN`, `Z:NNN`, `A:NNN`, `E:NNN` |
    | Event ID dictionary | None | 102-entry Table 40 |
    | Event-number prefix | Yes (line 1 starts with event number per Fig 6) | No (LCD-only) |
    | Baud | 4800 | 9600 |
    | RESET semantics | Active conditions survive | **Same** — active conditions survive |
    | ACK | Metadata, not log line | **Same** — Event 024/025 only |
    | Restore line | Bench-only evidence | Manual silent — bench dump required |

    Address tokenizer, verb dictionary, and separator diverge enough that a shared `VigilantParser` would be more conditional branches than shared code. Build a separate **`VigilantVsParser` class** covering VS1+VS2 jointly (mirrors the FireLite 9050+9600 pattern but with two parser CLASSES, not one shared class).

    **Rebuild plan (revised after VM + VS audits):**
    1. **Fix RESET semantics on VM-1** — switch from wipe-on-RESET fall-through to per-event `*_RST`-only clearing. Both manuals support this; no bench evidence needed to justify. **Could land as a standalone one-line fix in any (XX) build** without waiting for the full rebuild.
    2. **Add `PREALM` / `PRE-ALARM` / `PREALARM`** to VM-1 verb list as `FacpEventType.alarm` with prealarm severity (match Notifier convention).
    3. **Schedule a VM-1 bench-dump session** — capture full lifecycle (alarm/trouble/supervisory/monitor/prealarm/disable/test, RESET, SILENCE, ACK, operator commands). Save to `tti-helper-mobile/facp_manual/Vigilant/VM/bench_raw/`. Mirrors the (23) Notifier 3030 + (26) Fire-Lite 9600 dump pattern.
    4. **Schedule a VS1 or VS2 bench-dump session** — primary goal is capturing the (manual-undocumented) restore-line format. Save to `tti-helper-mobile/facp_manual/Vigilant/VS/bench_raw/`.
    5. **Add new `FacpModel` values** for VS family — `vigilantVS1` and `vigilantVS2` (one parser class covers both). Update brand-grouping picker.
    6. **Build `VigilantVsParser` class** — banner-driven against Table 39 + Table 40 dictionaries. Address tokenizer for `L:`/`D:`/`Z:`/`A:`/`E:` forms. Pipe-separator regex. Restore-line handling driven by bench-dump evidence.
    7. **Rewrite `vigilant_vm1_parser.dart` banner-driven** — bench-dump as primary evidence, manual as authority on RESET/PREALARM/address-format/operator-commands. Retain existing fixtures as regression tests.
    8. **Bench-verify on physical panels.**

    Cross-repo: `tti-helper-mobile` only. Slot recommendation: after (26) field-test closes out, alongside the parked Fire-Lite 9050 refactor — both are "manual-driven rebuild of a CloudWatch-calibrated parser" tasks of the same shape. Step 1 (RESET fix) is opportunistically extractable into any earlier build.

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
