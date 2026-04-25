# TTI Helper — Next Steps

Open items as of 2026-04-25, after landing:
- Mobile OTA + MQTT WiFi-change work (commits `320914e..ec5f35b` on `tti-helper-mobile/main`).
- Device ID gating + iOS theme polish (commit `345b155` on `tti-helper-mobile/main`).
- Troubleshooting Active page state-machine fix (commits `bca2d18..21d2050` on `tti-helper-mobile/main`).
  See `tti-helper-mobile/docs/TTI_HELPER_TROUBLESHOOTING.md` and `troubleshooting-active-bugs.pdf`.
- UX polish + sign-out confirmation (commit `cc2a684` on `tti-helper-mobile/main`, build 4).
- Notifier reset/normal differentiation (commits `78c0fe0..3a853ec` on `tti-helper-mobile/main`, build 5).

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(5) ready to upload

**Local state as of 2026-04-25:**
- `pubspec.yaml` is `1.0.0+5`. HEAD on `main` is `3a853ec`.
- `.ipa` is built and waiting at `tti-helper-mobile/build/ios/ipa/TTI Helper.ipa` (39 MB). **Not yet uploaded to App Store Connect.**

**Build history on App Store Connect:**
- (3) — Notifier CLR TB / SYSTEM NORMAL parser fix. Live in External, field-tested on user's iPhone (47 sessions, 0 crashes).
- (4) — adds version label on login screen + sign-out confirmation dialog. Uploaded 2026-04-25; user confirmed everything in (4) works in field test.
- (5) — adds SYSTEM NORMAL stays-visible behavior + SYSTEM RESET wipes alarms only (not troubles) + Notifier-only gating. **Local IPA ready**, awaiting upload.

**What (5) implements (Notifier-specific rules):**
- SYSTEM NORMAL → wipes everything + remains as the sole entry in Active until the next condition.
- SYSTEM RESET → wipes only alarms / ACTIVE-prefix events; troubles persist.
- Other panel models retain build-3 wipe-everything-on-reset behavior.

**Resumption checklist (in order):**
- [ ] Upload `1.0.0(5)` IPA via Transporter.
- [ ] Optional: in App Store Connect, expire build (4) so it's not confusable with (5).
- [ ] Assign (5) to Internal + External groups (auto-promotes — still 1.0.0 marketing version).
- [ ] Field-test 7-step sequence on real Notifier panel (TROUBL → CLR TB → ALARM → SYSTEM RESET → TROUBL → SYSTEM NORMAL → TROUBL).
- [ ] **CLR ACT pairing:** in build (4) field test, CLR ACT did not clear its matching ACTIVE event. Same code in (5). If reproduces, **capture the raw text of both messages** (gray monospace lines on the cards) — without those strings the parser/state-machine mismatch can't be diagnosed precisely. Likely a body-format difference (e.g. one has `Z003`, the other doesn't).
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at build time by `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission** (Apple reviewers reject placeholder launch images). Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.
- [ ] If 7-step Notifier test passes AND CLR ACT issue is fixed (likely build 6) AND launch image is replaced: bump to `1.0.1+1` for App Store Production submission.

**"What to Test" notes for the (5) External submission:**

> Build 1.0.0 (5) — Notifier Active page state-machine refinements.
> What's new since build (4): SYSTEM NORMAL wipes Active and stays visible there as the sole entry until the next event arrives; SYSTEM RESET clears alarms and ACTIVE-prefix events only (troubles persist); ACK/SIGNAL SILENCED have no effect on Active.
> Verify on Notifier panel: TROUBL → CLR TB pairing, ALARM → SYSTEM RESET clears alarm but troubles remain, SYSTEM NORMAL wipes + stays as sole entry, new TROUBL removes the SYSTEM NORMAL entry.
> Known issue under investigation: CLR ACT may not clear its matching ACTIVE event. If you see this, screenshot or copy the raw text of BOTH the ACTIVE line and the CLR ACT line.

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no more Export Compliance prompt on any build.

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
