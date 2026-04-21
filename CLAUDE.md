# CLAUDE.md — TTI Helper (Parent Workspace)

This is the **cross-project coordination layer** for the TTI Helper fire alarm monitoring system.
Use Claude from this directory when a task spans two or more repositories.
For single-project work, `cd` into the relevant subfolder and run Claude there.

---

## On session start

When Claude is invoked in this parent directory, begin every conversation with a brief status report:

1. Run `git status -s` and `git log @{u}..HEAD --oneline` inside each of `tti-helper-iot/`, `tti-helper-mobile/`, and `tti-helper-aws/` — summarize any uncommitted changes and unpushed commits per repo. If a repo is clean and in sync, say so in one line.
2. Read `TODO.md` (sibling to this file) and show the current next-steps list.
3. Keep the summary tight — one short section per repo, then the TODO list. No extra commentary until the user asks.

If the user immediately asks a specific question, answer that first and fold the status report into the response only if relevant.

---

## Repository Map

| Folder | Language | GitHub Repo | Purpose |
|--------|----------|-------------|---------|
| `tti-helper-aws/` | TypeScript (CDK) | tti-helper-aws | AWS infrastructure: Cognito, IoT Core, IAM |
| `tti-helper-mobile/` | Dart (Flutter) | tti-helper-mobile | iOS/Android app: alarms, commands, BLE provisioning |
| `tti-helper-iot/` | C (FreeRTOS) | tti-helper-iot | ESP32 firmware: reads FACP serial, publishes MQTT |
| `tti-helper-hw/` | Gerber / PCB | tti-helper-hw | Hardware design: PCB gerbers, drill files, board analysis |
| `tti-helper-legal/` | HTML | tti-helper-legal | Public legal pages served via GitHub Pages at `https://aavalosg.github.io/tti-helper-legal/` (privacy policy) |
| `tti-helper-portal-admin/` | TBD (Phase 3, planning) | TBD | Internal back-office console for TTI staff. Empty placeholder until Phase 3 starts. See `PHASE_3_PLAN.md`. |
| `tti-helper-portal-client/` | TBD (Phase 3, planning) | TBD | Multi-tenant customer portal (pool model, tenant_id). Empty placeholder until Phase 3 starts. See `PHASE_3_PLAN.md`. |
| `../ttireader/` (sibling) | C (Arduino) | ttireader | Legacy ESP32 firmware — pre-refactor ancestor of `tti-helper-iot`. Sibling to this parent folder, read-only reference. |

Each subfolder is an independent git clone with its own GitHub remote. This parent folder provides shared architectural context only — it does not track the subfolders via git submodules.

---

## System Architecture

```
FACP (RS232)
  └─► ESP32 — tti-helper-iot (C / FreeRTOS)
                │  MQTT over TLS / X.509 cert
              AWS IoT Core — tti-helper-aws (CDK / us-east-2)
                │  MQTT over WebSocket / SigV4 + Cognito credentials
              Flutter App — tti-helper-mobile (Dart)
```

- **Device → Cloud**: X.509 certificate auth; ESP32 publishes to `dt/` topics
- **Mobile → Cloud**: Cognito OTP → Identity Pool → STS credentials → MQTT over WSS port 443
- **Cloud → Device**: Mobile publishes to `cmd/` topics; ESP32 subscribes
- **BLE**: Used **only** for initial WiFi provisioning; not used at runtime

---

## MQTT Topics (Source of Truth)

> **Do not change topic strings without updating all affected repos.**
> Cross-repo impact is listed in the table below.

| Topic | Direction | Publisher | Subscriber |
|-------|-----------|-----------|------------|
| `dt/esp32/ttireaderv1/{thingID}` | Device → Cloud | tti-helper-iot | tti-helper-mobile |
| `dt/esp32/ttireaderv1/{thingID}/status` | Device → Cloud | tti-helper-iot | tti-helper-mobile |
| `cmd/esp32/ttireaderv1/{thingID}` | Cloud → Device | tti-helper-mobile | tti-helper-iot |

### Alarm payload (`dt/esp32/ttireaderv1/{thingID}`)

```json
{
  "thingID": "78E36D5EED84",
  "message": "<raw FACP UART line>",
  "seqnbr": "<ISO-8601 timestamp>"
}
```

Published on every FACP UART message, and once on AWS IoT connect with `message = "TTI connected"`.

### Status payload (`dt/esp32/ttireaderv1/{thingID}/status`)

```json
{
  "thingID": "78E36D5EED84",
  "firmware_version": "1.0.3",
  "timestamp": "<ISO-8601>"
}
```

Published once on AWS IoT connect. Used by the mobile Device Management screen to display current firmware version.

### Command payload (`cmd/esp32/ttireaderv1/{thingID}`)

```json
{
  "thingID": "78E36D5EED84",
  "command": {
    "name": "<COMMAND_NAME>",
    "type": "RELAY | UART",
    "action": "<UART bytes — only for UART type>"
  }
}
```

| Command | name | Behavior |
|---------|------|----------|
| Acknowledge | `ACK` | Pulses ACK relay (or writes `~L` to UART) |
| Signal Silence | `SILENCE` | Pulses SILENCE relay (or writes `~M` to UART) |
| System Reset | `RESET` | Pulses RESET relay (or writes `~N` to UART) |
| Update WiFi | `WIFI` | Saves new SSID/password to NVS, restarts |
| Reconfigure UART | `SETUP_UART` | Changes baud rate, pins |
| OTA update | `OTA` | Downloads firmware `.bin` from S3, reboots |

> **MQTT buffer limit:** 1024 bytes. Use short direct S3 URLs for OTA — pre-signed URLs (~700+ chars) exceed the limit.

---

## Device Identity

The ESP32's device identifier is its Wi-Fi MAC address with colons removed, uppercased.

```
Format:  <HEX12>
Example: 78E36D5EED84
```

Used as: AWS IoT Thing Name, MQTT Client ID, MQTT topic suffix, BLE device name suffix.

---

## BLE Provisioning Contracts

Used only for initial WiFi credential push. Once WiFi connects, BLE exits.

| Property | Value |
|----------|-------|
| Device name | `TTIReader-<MAC_LAST4>` (e.g. `TTIReader-ED84`) |
| Service UUID | `4fafc201-1fb5-459e-8fcc-c5c9c331914b` |
| Characteristic UUID | `beb5483e-36e1-4688-b7f5-ea07361b26a8` |

**Command protocol** — mobile writes `<CMD>:<value>` to the characteristic:

| Cmd | Purpose |
|-----|---------|
| `A:<pin>` | Authenticate session (required if PIN set in NVS) |
| `S:<ssid>` | Set WiFi SSID |
| `P:<password>` | Set WiFi password → triggers save + restart |

---

## AWS Resources (us-east-2)

| Resource | Value |
|----------|-------|
| Region | `us-east-2` |
| IoT Endpoint | `a287nkw2rti5xe-ats.iot.us-east-2.amazonaws.com` |
| Cognito User Pool | `us-east-2_7rlY1Cpti` |
| Cognito App Client | `2it4q8ihaluj8ebli0bq31jg3m` |
| Cognito Identity Pool | `us-east-2:842c023e-3ddd-4710-9433-f89352259496` |
| IoT Policy (device) | `tti-iot-device-policy` |
| IoT Policy (mobile) | `tti-mobile-iot-policy` |

### Mobile IAM permissions (CognitoAuthRole)

```
iot:Connect    → arn:aws:iot:us-east-2:*:client/tti-mobile-*
iot:Subscribe  → arn:aws:iot:us-east-2:*:topicfilter/dt/esp32/ttireaderv1/*
iot:Receive    → arn:aws:iot:us-east-2:*:topic/dt/esp32/ttireaderv1/*
iot:Publish    → arn:aws:iot:us-east-2:*:topic/cmd/esp32/ttireaderv1/*
```

> If you add a new topic prefix (e.g. a status topic), update the IAM policy in `tti-helper-aws/lib/tti-helper-aws-stack.ts`.

---

## Deployment Order (from scratch)

1. **`tti-helper-aws/`** — `npx cdk deploy`
2. Copy CDK stack outputs into `tti-helper-mobile/lib/core/config/aws_config.dart`
3. Flash **`tti-helper-iot/`** firmware via PlatformIO; provision device via BLE from the mobile app

---

## Cross-Repo Impact Table

| What changes | Repos that must also change |
|---|---|
| MQTT topic strings | `tti-helper-aws` (IoT policy), `tti-helper-mobile` (subscriptions) |
| Alarm JSON field names (`thingID`, `message`, `seqnbr`) | `tti-helper-mobile` (parsing), `tti-helper-aws` (IoT rules if added) |
| Status JSON field names (`firmware_version`) | `tti-helper-mobile` (Device Management screen) |
| Command names (`RESET`, `SILENCE`, `ACK`, `OTA`, `WIFI`, `SETUP_UART`) | `tti-helper-mobile` (UI actions), `tti-helper-aws` (Lambda dispatcher if added) |
| BLE Service / Characteristic UUIDs | `tti-helper-mobile` (`lib/core/ble/ble_config.dart`) |
| BLE device name prefix (`TTIReader-`) | `tti-helper-mobile` (device discovery filter) |
| Device identity format (MAC no colons) | `tti-helper-aws` (Thing names, topic routing), `tti-helper-mobile` (display) |
| AWS region or IoT endpoint | `tti-helper-mobile` (`aws_config.dart`), `tti-helper-iot` (secrets header) |
| Cognito pool IDs | `tti-helper-mobile` (`aws_config.dart`) |

---

## Key Files per Repo

| File | Role |
|------|------|
| `tti-helper-aws/lib/tti-helper-aws-stack.ts` | Single CDK stack — source of truth for all AWS resource IDs |
| `tti-helper-mobile/lib/core/config/aws_config.dart` | Receives CDK output values post-deploy |
| `tti-helper-mobile/lib/core/ble/ble_config.dart` | BLE UUIDs used by the provisioning flow |
| `tti-helper-iot/doc/CONTRACTS.md` | Authoritative firmware communication contracts |
| `tti-helper-iot/include/ttireader_secrets.h` | Device secrets (not committed; template in repo) |
| `tti-helper-mobile/docs/TECHNICAL_ARCHITECTURE.md` | Full mobile + AWS technical architecture |
| `tti-helper-mobile/docs/TTI_HELPER_FULL.md` | Full product specification |
