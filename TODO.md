# TTI Helper — Next Steps

Open items as of 2026-04-19, after landing the mobile OTA + MQTT WiFi-change work (commits `320914e..ec5f35b` on `tti-helper-mobile/main`).

## 1. Verify AWS IoT policy matches new status topicfilter

Mobile now subscribes to `dt/esp32/ttireaderv1/+/status` (wildcard) instead of the per-thingID form. Confirm the deployed policy in `tti-helper-aws/lib/tti-helper-aws-stack.ts` isn't stricter — the existing `topicfilter/dt/esp32/ttireaderv1/*` looks compatible, but the previous exact-match subscription was being silently denied, so double-check what's actually live.

**Repo:** `tti-helper-aws`

## 2. End-to-end test the two new mobile flows

Neither has been exercised on real hardware yet:

- **MQTT-based WiFi change** (`wifi_update_mqtt_sheet.dart`): device receives `WIFI` command while online, saves creds, reboots onto new network. Verify the 60s reconnect timeout behavior.
- **OTA via S3 manifest**: mobile compares device-reported `firmware_version` against `latest.json`, publishes `OTA` command with firmware URL. Verify the device downloads, flashes, and comes back up.

**Repos:** `tti-helper-mobile`, `tti-helper-iot` (device side)

## 3. Address risks flagged in the mobile review

In priority order:

1. **OTA feedback loop** — user gets no confirmation the device accepted or completed the update. Consider emitting an OTA status/progress event on the status topic.
2. **Firmware URL validation** — no check that the manifest URL is reachable or well-formed before sending OTA command.
3. **Status cache invalidation** — `MqttService._latestStatusByThingId` is never cleared; goes stale if a device's firmware changes out-of-band.
4. **BLE password-write try/catch** — currently swallows all exceptions. Acceptable given the firmware reboot race, but worth distinguishing expected disconnect from other failures.
5. **Wildcard status subscription scope** — `+/status` subscribes to all devices; fine for single-device users, may need tightening later.
