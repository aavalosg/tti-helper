# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(8) ready to upload

**Local state as of 2026-04-26:**
- `pubspec.yaml` is `1.0.0+8`. HEAD on `tti-helper-mobile/main` is `af61122`.
- (8) IPA built and waiting at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa`. **Not yet uploaded to App Store Connect** — overwrites the (7) IPA.

**Commits landed for (8):**
- `5e6c270` — `fix(inspection): keep notes scoped to each event entry`
- `e1149a9` — `feat(reports): drop TYPE/ZONE columns, photo appendix, add sort options`
- `eb1e11e` — `fix(firelite): make RESET / ACK / SILENC no-ops on Active`
- `af61122` — `fix(reports): keep per-event photos inline in Inspection report` (follow-up; `e1149a9` had inadvertently dropped per-event photos along with the appendix)

**Build history on App Store Connect:**
- (3) — Notifier CLR TB / SYSTEM NORMAL parser fix. Field-tested (47 sessions, 0 crashes).
- (4) — version label on login screen + sign-out confirmation dialog. Field-tested.
- (5) — SYSTEM NORMAL stays-visible + SYSTEM RESET wipes alarms only + Notifier-only gating. Field-tested.
- (6) — skipped.
- (7) — Active-tab bucket filter, ACTIVE→Others, Inspection bucket filter, Inspection photos in report PDF, 2400/4800 baud picker, CLR ACT concatenation fix. Field-tested 2026-04-26.
- (8) — Inspection notes scoping + report cleanup (no TYPE/ZONE cols, no Photo Appendix, sort options) + Fire-Lite RESET no-op. **Local IPA ready**, awaiting upload.

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no Export Compliance prompt on any build.

**Outstanding pre-Production gate (still not done):**
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at every `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission**. Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.

**Resumption checklist for (8):**
- [ ] Upload `1.0.0(8)` IPA via Transporter.
- [ ] Optional: in App Store Connect, expire build (7) so it's not confusable with (8).
- [ ] Assign (8) to Internal + External groups.
- [ ] Field-test on bench:
  - Inspection: switch between event entries with notes filled — confirm prior device's notes don't leak into the next.
  - Inspection report PDF: confirm Photo Appendix gone, TYPE / FACP ZONE columns gone, sort selector defaults to Address/Device with Time available.
  - Troubleshooting report PDF: confirm TYPE / ZONE columns gone, by-time order only.
  - Fire-Lite RESET on the panel: confirm Active list does NOT wipe; events persist until their CLEAR(x) arrives.
  - Fire-Lite NAC: trigger NAC 1 OPEN trouble + CLEART pair — confirm OPEN clears, NAC 1 FAULT (if present) remains active.

**"What to Test" notes for the (8) External submission:**

> Build 1.0.0 (8) — Inspection + Reporting cleanup, Fire-Lite RESET fix.
> Inspection: notes are now scoped per event entry — switching between devices in a testing area shows each device's own notes only.
> Reports: TYPE and FACP ZONE columns removed from Troubleshooting and Inspection reports. Inspection report drops the trailing Photo Appendix; zone-level and per-event photos now render inline within each testing-area block. Inspection report gains a sort selector (Address/Device default, Time as alternate). Troubleshooting report stays in time order.
> Fire-Lite: pressing RESET on the panel no longer wipes Active in the app; events clear only via their own CLEAR(x) / RESTOR commands, per panel spec.

## 0b. Done in (8)

- [x] **Inspection — "Comments by Device" carryover bug.** Stable `ValueKey(entry.id)` on `_EventEntryTile` prevents Flutter from reusing controller state across entries when the bucket filter changes or list reorders. (`5e6c270`)
- [x] **Reports — drop Photo Appendix from Inspection report.** (`e1149a9` removed the appendix; `af61122` restored per-event photos inline beneath each zone's data table, captioned by event time + description, after I noticed the appendix removal had also dropped per-event photos by mistake.)
- [x] **Reports — drop TYPE and FACP ZONE columns** from Troubleshooting and Inspection reports. (`e1149a9`)
- [x] **Reports — sort options.** Troubleshooting keeps only by-time (no selector); Inspection gains an `InspectionReportSort` selector with byAddressDevice (default, sorts entries by description → time → result within zone) and byTime. Threaded through `FinalizeResult` → `InspectionPdfPreviewArgs` → generator. (`e1149a9`)
- [x] **Fire-Lite — RESET / ACK / SILENC are no-ops on Active.** Early-return on `FacpModel.firelite9050` in `_updateActiveMap`'s system case. Vigilant / EST iO / unknown panels keep the legacy wipe behavior. (`eb1e11e`)
- [x] **Fire-Lite — CLEART NAC pairing verified via tests.** Parser already produces matching `incidentKey` for TROUBL/CLEART NAC OPEN; OPEN/FAULT keys are distinct. The reported "CLEART NAC isn't working" symptom on the bench was the RESET wipe arriving first — addressed by the RESET no-op above. New tests cover NAC OPEN pairing and OPEN-vs-FAULT key disjointness. (`eb1e11e`)

## 0c. Backlog — not bound to (8)

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
