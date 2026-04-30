# TTI Helper — Next Steps

> **See `ROADMAP.md`** for the full Phase 1–4 plan (troubleshooting, inspection, portal admin, record of completion). This file is the immediate next-steps; ROADMAP is the durable phase plan.

## 0. In flight — TestFlight 1.0.0(14) ready to build/upload

**Local state as of 2026-04-30:**
- `pubspec.yaml` is `1.0.0+14`. HEAD on `tti-helper-mobile/main` is `39a882d` (4 prior commits unpushed: `35e76a5`, `eb56572`, `610eba1`, `fd238e0`).
- (14) IPA **not yet built** — `flutter build ipa` will overwrite the (13) IPA on disk. Build before next Transporter upload.

**Commits landed for (14):**
- `313455f` — `fix(estio): DSBL/TEST enter active map; accept S+H multi-criteria code`
- `39a882d` — `chore: bump build to 1.0.0+14`

**Build history on App Store Connect (recent):**
- (10) — History-page event ordering under panel burst; firmware v1.0.7 OTA companion (UART 256 B → 4 KB drain-per-tick). Field-tested.
- (11) — Hybrid EST iO pairing (wrong direction — superseded by (12)).
- (12) — EST iO pairing direction corrected; `descriptionFirst => false` for all models. Field-tested 2026-04-29.
- (13) — Bundled: `_classify()` Battery Missing/Power Loss/etc descriptor pairing + full EST iO parser rewrite (17 STATUS codes, 4 address formats, iO64/iO500 + iO64/iO1000 split) + one-shot `estIO1000` → `estIO500` SharedPrefs migration. Released + field-tested 2026-04-29..30. Two issues surfaced (both addressed in (14)).
- (14) — DSBL/TEST mapping fix + S+H multi-criteria detector code. **IPA not yet built**; awaiting upload.

**Permanent fixes already in place (won't repeat):**
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` — no Export Compliance prompt on any build.

**Outstanding pre-Production gate (still not done):**
- [ ] Replace iOS launch image — currently the default Flutter placeholder (flagged at every `flutter build ipa`). Not blocking for TestFlight, but **required before App Store Production submission**. Asset lives at `ios/Runner/Assets.xcassets/LaunchImage.imageset/`.

**Resumption checklist for (14):**
- [ ] `flutter build ipa` (overwrites (13) on disk).
- [ ] Upload `1.0.0(14)` IPA via Transporter.
- [ ] Assign (14) to Internal + External groups.
- [ ] Field-test on EST iO bench:
  - Trigger a Smoke + Heat detector at `L:1 D:001` — `S+H ACT` should appear in Active under the **Alarm** bucket with description "Smoke + Heat" (or the panel's custom descriptor if line2 is supplied). After `S+H RST`, the row should leave Active.
  - Trigger DSBL on a device (disable a pull station) — `DSBL ACT` should appear in Active under **Others**. After re-enabling (`DSBL RST`), the row should leave Active.
  - Run a walk test on a device — `TEST ACT` should appear in Active under **Others**. After test ends (`TEST RST`), it should clear.
  - Sanity: existing flows (MON ACT/RST, ALRM, TRBL, SUPV, COSU, SYSTEM NORMAL via `MON RST + E:020`) should behave exactly as in (13).

**"What to Test" notes for the (14) External submission:**

> Build 1.0.0 (14) — two EST iO parser fixes from (13) field test.
> Smoke + Heat (S+H) detector events: Trigger a multi-criteria smoke+heat detector on the EST iO panel. `S+H ACT` events now appear in Active under the Alarm bucket (previously they fell through as raw SYSTEM rows because the parser regex required letters-only STATUS codes). The matching `S+H RST` clears the row.
> Disable / Walk Test events: Disable a device (DSBL) or put a device into walk test (TEST). Both now appear in Active under the Others bucket and clear when their RST event arrives. Previously they were classified as system events and never entered Active, so their RST had nothing to clear.

## 0b. Done in (14)

- [x] **DSBL / TEST → FacpEventType.active.** Both STATUS codes previously mapped to `FacpEventType.system`, which excludes them from `_updateActiveMap`. Now they enter the active map under the device address (`L:N D:NNN`) so the matching RST clears them via the standard zone-based RST flow. Bucket stays Others. (`313455f`)
- [x] **S+H multi-criteria detector code.** Header regex broadened from `[A-Z]{3,4}` to `[A-Z][A-Z+]{2,4}` so `S+H` matches; added to alarm class with label "Smoke + Heat". Future undocumented multi-criteria codes will at least surface their descriptor instead of dumping the raw line. (`313455f`)
- [x] **Pubspec bumped to `1.0.0+14`.** (`39a882d`)
- [x] Tests: 41 EST iO tests pass (was 39 — added S+H ACT/RST). Full mobile suite: 171/171.

## 0c. Backlog — not bound to (14)

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
