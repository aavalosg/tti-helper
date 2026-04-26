# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(9) ready to upload

**Local state as of 2026-04-26:**
- `pubspec.yaml` is `1.0.0+9`. HEAD on `tti-helper-mobile/main` is `4b477dc`.
- (9) IPA built and waiting at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa` (39 MB). **Not yet uploaded to App Store Connect** — overwrites the (8) IPA.

**Commits landed for (9):**
- `4deb432` — `fix(firelite): pair TROUBL/CLEARt across trailing state qualifiers`
- `0b9ea45` — `feat(reports): restore ADDRESS column in event tables`
- `6674ae5` — `fix(session): clear active events on endSession`
- `4b477dc` — `chore: bump build to 1.0.0+9`

**Build history on App Store Connect:**
- (3) — Notifier CLR TB / SYSTEM NORMAL parser fix. Field-tested (47 sessions, 0 crashes).
- (4) — version label on login screen + sign-out confirmation dialog. Field-tested.
- (5) — SYSTEM NORMAL stays-visible + SYSTEM RESET wipes alarms only + Notifier-only gating. Field-tested.
- (6) — skipped.
- (7) — Active-tab bucket filter, ACTIVE→Others, Inspection bucket filter, Inspection photos in report PDF, 2400/4800 baud picker, CLR ACT concatenation fix. Field-tested 2026-04-26.
- (8) — Inspection notes scoping + report cleanup (no TYPE/ZONE cols, no Photo Appendix, sort options) + Fire-Lite RESET no-op. Uploaded + field-tested 2026-04-26 — three issues surfaced, all addressed in (9).
- (9) — Fire-Lite NAC pairing across trailing state qualifiers + ADDRESS column restored in reports + Active Events cleared on session end. **Local IPA ready**, awaiting upload.

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no Export Compliance prompt on any build.

**Outstanding pre-Production gate (still not done):**
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at every `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission**. Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.

**Resumption checklist for (9):**
- [ ] Upload `1.0.0(9)` IPA via Transporter.
- [ ] Optional: in App Store Connect, expire build (8) so it's not confusable with (9).
- [ ] Assign (9) to Internal + External groups.
- [ ] Field-test on bench:
  - Fire-Lite NAC: trigger NAC 1 fault — TROUBL `NAC 1 FAULT` should appear in Active. Clear it — CLEARt `NAC 1 OPEN` should arrive AND remove the TROUBL from Active (the (8) bug). Repeat with NAC 2 to confirm cross-circuit isolation.
  - Fire-Lite system events (battery, ground, comm): trigger trouble + clear — pairing should hold across any trailing-word mismatch.
  - Generate a Troubleshooting PDF — every event row should show an ADDRESS column with the device address (e.g. `L:1 D:23`, `E:042`, or `-`).
  - Generate an Inspection PDF — ADDRESS column should appear in Zone Event Table and Event Log.
  - Exit a Troubleshooting session via End Session sheet **without logging out**, start a new session — Active Events list should be empty (no leftover events from prior session).

**"What to Test" notes for the (9) External submission:**

> Build 1.0.0 (9) — three field-test fixes from (8).
> Fire-Lite NAC clearing: Trigger a NAC fault on the panel (open NAC 1 circuit), then clear it. The trouble in Active Events now disappears when the panel emits its CLEARt. Previously the trouble lingered because Fire-Lite uses different trailing words across a paired event ('FAULT' for trouble, 'OPEN' for its restore). Verify with NAC 1, NAC 2, and other system-level events (battery, ground, comm).
> Reports — Address column: Generate a Troubleshooting PDF and an Inspection PDF. Every event row now shows an ADDRESS column (Device ID like L:1 D:23 / E:042, or '-' when no address is reported). The column was inadvertently removed in the (8) report cleanup; this restores it. TYPE column stays removed.
> Active Events reset between sessions: Exit a Troubleshooting session via the End Session sheet without logging out, then start a new Troubleshooting session. Active Events should start empty.

## 0b. Done in (9)

- [x] **Fire-Lite NAC pairing across trailing state qualifiers.** `_incidentKey()` fallback now uses `_systemComponentKey` (port of EST iO pattern) — `IN SYSTEM NAC 1 FAULT` and `IN SYSTEM NAC 1 OPEN` both map to `sys:NAC 1`. Same fix covers FAULT/OPEN/SHORT variants of any system-level Fire-Lite event. Tests: dropped obsolete OPEN-vs-FAULT-distinct assertion; added FAULT-paired-with-OPEN, NAC 1 ≠ NAC 2, and NO BATTERY regression. (`4deb432`) Note: this corrects (8)'s incomplete fix — the (8) `eb1e11e` analysis assumed the bench symptom was just the RESET wipe arriving first; field test of (8) showed the FAULT/OPEN trailing-word mismatch is real.
- [x] **Reports — ADDRESS column restored** in Troubleshooting Event Table, Inspection Zone Event Table, and Inspection Event Log. 48px fixed width, after TIME, before DESCRIPTION. Renders `e.zone ?? '-'` (or `e.facpZone ?? '-'` for inspection zone entries). TYPE column stays removed. (`0b9ea45`)
- [x] **Active Events clear on `endSession()`.** `alarmProvider` (global Riverpod AsyncNotifier) is invalidated when a Troubleshooting session ends, triggering existing `onDispose` cleanup (clears `_activeMap`, `_buffer`, cancels MQTT subscription). Next session starts with a fresh build. (`6674ae5`)
- [x] **Pubspec bumped to `1.0.0+9`.** (`4b477dc`)

## 0c. Backlog — not bound to (9)

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
