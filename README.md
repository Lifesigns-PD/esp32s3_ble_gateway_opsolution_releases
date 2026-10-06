# ESP32-S3 BLE Gateway (OP solution)

ESP-IDF firmware for an ESP32-S3 that acts as a **BLE-to-HTTP gateway** for Lifesigns medical devices. It scans for supported devices, connects up to four of them at the same time (one per slot), decodes each device's BLE protocol, and posts the readings as JSON over WiFi to a configurable HTTP or HTTPS endpoint.

- **Target:** ESP32-S3 (N16R8: 16 MB flash, 8 MB octal PSRAM)
- **Framework:** ESP-IDF 6.1, NimBLE host
- **Firmware version:** set in the top-level `CMakeLists.txt` (see [Versioning](#versioning))

## Supported devices

Each device type has its own connection slot and its own upload URL.

| Slot | Name | Device | Matched by | Data posted |
|---|---|---|---|---|
| 0 | `ECG` | **ezFlex** (Lepu ER1) or **ezBelt** (Frontier X2) | name `ER1…` / `Frontier…` | 125-sample `livecg-data` ECG chunks with HR, battery, posture |
| 1 | `LS06` | **PC-100** BP & SpO2 sensor | name `PC-100…` **and** service `FFF0` | Live Nexus JSON: SpO2, PR, PI, pleth wave; BP result or error when measured |
| 2 | `LS_TEMP` | **Tracky** IR thermometer | name `My Thermometer` | One temperature reading per measurement (°C and °F) |
| 3 | `LS_BODYWH` | **BF1H** height + weight scale (AiLink) | name `BF1H…`, or service `FFE0` + AiLink CID `0x0026` | Weight, height and BMI when the measurement finishes |

- Only one device per slot is connected at a time.
- When all four slots are connected, scanning pauses to leave radio time for the links and WiFi. It resumes as soon as a slot is free.
- **AUTO mode** (default): each slot connects the first matching device in range.
- **MANUAL mode:** each slot connects only the one MAC address assigned to it through the BLE config mode. A slot without a MAC stays empty.

## How it works

```
 power on ──► BLE config mode (60 s, phone app) ──► restart ──► normal mode
                                                                 │
   ┌─────────────────────────────────────────────────────────────┘
   ▼
 BLE scan ──► connect (slot 0-3) ──► GATT handshake ──► notifications
                                                          │ decode per device
                                                          ▼
                       JSON ──► PSRAM queue ──► HTTP(S) POST to the slot's URL
                                                (retry with backoff on failure)
```

- **Config mode** runs only after a real power-off/on (not after a crash, watchdog, OTA or software restart). It advertises as `GATEWAY-OP_XXXXXX` (last 6 hex digits of the WiFi MAC) for 60 s, extended while a phone stays connected, then restarts into normal mode. See [BLE configuration](#ble-configuration).
- **Normal mode** joins WiFi, syncs time over SNTP and runs the BLE central and the uploader.
- **Uploads** are queued in PSRAM (default 1024 payloads). Network errors and HTTP 408/429/5xx are retried indefinitely with exponential backoff (1 s up to 30 s); other 4xx responses drop the payload. The oldest payload is dropped if the queue or PSRAM fills up.
- **Self-healing BLE:** a watchdog cancels stuck connection attempts, frees slots whose link vanished silently, and restarts a silent scanner.

## Device protocols

| Device | Connection | Notes |
|---|---|---|
| ezFlex (ER1) | Notify on the Lepu characteristic | A5-framed packets with CRC-8/ATM. Legacy firmware `0.1.0` is polled every 900 ms. Samples are converted to µV and linearly detrended. Serial number is `F` + last 6 hex digits of the MAC |
| ezBelt (Frontier X2) | LED-off command, then notify | Packet type 0x0D: 100 ECG samples + HR + posture. Type 0x1B: battery |
| PC-100 | Notify on `FFF1`; info query `AA 55 51 02 01 C8` on `FFF2` at connect and every 5 min | 0x53 SpO2/PR/PI, 0x52 pleth, 0x43 BP result/error, 0x51 battery/version. No BP start command is sent: BP is decoded when the user measures on the device. Device stays connected |
| Tracky | Notify on `FFF3` | 5-byte frames `AA b1 hi lo sum`, temperature × 100. °C or °F inferred from the range; repeated frames within 3 s are posted once |
| BF1H | Notify on `FFE2` (A7) and `FFE3` (A6) | TEA handshake (answers the scale's challenge), A6 version query, A7 status. A7 payloads are XOR-encrypted with the advertised MAC + CID on module firmware after 2019-08-13. Height may arrive after the "finished" message; the gateway waits up to 4 s for it |

## Uploads

Every slot posts to its own full URL. By default all four slots post to the local server:

```
http://172.16.22.191:8080/
```

- `http://` (plain, e.g. a local server on the LAN) and `https://` (TLS with the ESP-IDF CA bundle) are both supported.
- Change the factory URLs in menuconfig → **Upload endpoints**, or per gateway from the phone app with `set_url`.

### JSON payloads

| Slot | `deviceType` / shape | Key fields |
|---|---|---|
| ECG | JSON array with one `livecg-data` object | `macID`, `serialNo`, `data[125]`, `heartrate`, `battery`, `packetNo`, `startTime`/`endTime`, `posture`, `timestamp` |
| LS06 | Nexus object | `deviceID` (`LS06` + last 4 hex of MAC), `bp{…}`, `spo2{…}`, `pleth.plethWave[]`, `device{…}` |
| LS_TEMP | `"deviceType":"LS_TEMP"` | `deviceID` (`TK` + MAC), `temperature_C` / `temperature_F` (×100), `deviceUnit`, `epochTime`, `seq` |
| LS_BODYWH | `"deviceType":"LS_BODYWH"` in `device` | `measurement{weight (kg), weightStable, height (cm), bmi, bmiCategory}`; values the scale did not report are `"N/A"` |

## BLE configuration

The phone app configures the gateway during the config-mode window over one GATT service:

| | UUID |
|---|---|
| Service | `6c730001-1f2a-4d8e-9b3c-6a5f0e7d2c10` |
| WRITE (commands) | `6c730002-1f2a-4d8e-9b3c-6a5f0e7d2c10` |
| READ / Notify (config, acks, events) | `6c730003-1f2a-4d8e-9b3c-6a5f0e7d2c10` |

Messages are newline-terminated JSON. Commands:

| Command | Purpose |
|---|---|
| `ping`, `get_config` | Liveness, full settings |
| `set_wifi` | WiFi SSID and password |
| `set_url` | Full POST URL for one slot |
| `set_mode` | `AUTO` or `MANUAL` |
| `set_slot_mac`, `clear_slot_mac` | Manual mode: the one device MAC per slot |
| `reset_config` | Factory defaults |
| `ota` | Download and install firmware from the OTA URL |
| `finish` | Apply and restart into normal mode |

Every command is acknowledged, and settings are saved to flash immediately. The complete protocol for the mobile app (framing, every field, error codes, OTA events) is in [docs/BLE_CONFIG_PROTOCOL.md](docs/BLE_CONFIG_PROTOCOL.md).

## Build and flash

```bash
. ~/.espressif/tools/activate_idf_v6.1.sh   # or your ESP-IDF 6.1 export script
idf.py set-target esp32s3                   # first time only
idf.py menuconfig                           # optional: Lifesigns BLE Gateway menu
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

- The first time on a board that ran other firmware, run `idf.py -p /dev/ttyUSB0 erase-flash` before flashing (the partition layout differs).
- After pulling changes that add Kconfig options, run `idf.py reconfigure` (or a full clean) before building. Otherwise a stale `sdkconfig` can fail with `CONFIG_GW_… undeclared`.
- `sdkconfig` is not committed. A fresh clone builds its configuration from `sdkconfig.defaults*` and the Kconfig defaults.

### Partition table (16 MB)

| Name | Offset | Size |
|---|---|---|
| nvs | 0x9000 | 24 KB |
| otadata | 0xF000 | 8 KB |
| phy_init | 0x11000 | 4 KB |
| ota_0 | 0x20000 | 6 MB |
| ota_1 | 0x620000 | 6 MB |
| storage (spiffs) | 0xC20000 | 3.9 MB |

## Configuration (menuconfig → Lifesigns BLE Gateway)

| Menu | Option | Default |
|---|---|---|
| BLE config mode | `GW_CONFIG_MODE_ENABLE` / `GW_CONFIG_WINDOW_SEC` / `GW_CONFIG_MAX_EXTEND_SEC` | on / 60 s / +120 s |
| | `GW_OTA_URL` | GitHub release `ota_update/firmware.bin` |
| | `GW_OTA_CHECK_PROJECT_NAME` | on (rejects images built for another project) |
| WiFi | `GW_WIFI_SSID` / `GW_WIFI_PASSWORD` | factory WiFi (overridden by `set_wifi`) |
| Upload endpoints | `GW_URL_ECG` / `GW_URL_LS06` / `GW_URL_TEMP` / `GW_URL_BODYWH` | `http://172.16.22.191:8080/` |
| | `GW_POST_QUEUE_LEN` / `GW_PSRAM_MIN_FREE_KB` | 1024 / 1024 KB |
| | `GW_RETRY_MAX_BACKOFF_MS` / `GW_HTTP_TIMEOUT_MS` | 30000 / 10000 |
| | `GW_NEXUS_MIN_POST_MS` | 1000 (PC-100 live post rate limit) |
| BLE | `GW_MIN_RSSI` | -95 dBm |
| | `GW_DATA_TIMEOUT_SEC` | 15 s (ECG belts only) |
| | `GW_RECONNECT_COOLDOWN_SEC` | 10 s |
| | `GW_BF1H_HEIGHT_WAIT_MS` | 4000 |
| | `GW_BF1H_USER_SEX` / `_AGE` / `_HEIGHT_CM` | 1 / 30 / 0 (sent only if the scale asks) |
| | `GW_PC100_INFO_INTERVAL_SEC` | 300 |
| General | `GW_STATUS_INTERVAL_SEC` / `GW_SNTP_SERVER` | 10 s / `pool.ntp.org` |

## Logs

Every 10 s the gateway prints a status block (tag `BLE_CENT`):

```
Status | Host: GATEWAY-OP_4C7DFC | FW: 1.0.0 | WiFi: Connected | SSID: … | IP: 172.16.22.207 | HTTP: OK (200) | Mode: AUTO | Sent: 281 | Queued: 0/1024 | Retries: 0 | Dropped: 0 |
Heap   | Internal: 135303 free (min 96524, largest 61440) | PSRAM: 8361628 / 8388608 free
  -> [Slot 0] CONNECTED | ID: f2:4a:4b:a3:e9:e3 | Type: ezFlex | SN: FA3E9E3 | Bat: 100% | Blepkts: 21 | Post: 10 | RSSI: -83
  -> [Slot 3] CONNECTED | ID: 01:b6:ec:eb:07:10 | Type: LS_BODYWH | SN: BMEB0710 | Bat: 100% | Blepkts: 14 | Post: 0 | RSSI: -64
  -> [Scan] Scanning | 28 BLE device(s) in range
       ER1-L 0430         | ID: e7:23:ae:74:0a:d7 | RSSI: -75 | seen 3s ago | idle
```

- `Blepkts` and `Post` count frames and successful posts since the previous status block.
- Disconnects are logged with the reason, the connection time and the age of the last data.

## Testing with the local server

`test/local_server.py` is a dependency-free receiver for the gateway's posts. Run it on the PC at `172.16.22.191`:

```bash
python3 test/local_server.py                 # port 8080, summary + JSON per post
python3 test/local_server.py --quiet         # one summary line per post
python3 test/local_server.py --full          # all ECG / pleth samples
python3 test/local_server.py --log rx.jsonl  # also save every payload
```

It answers `200`, recognises each payload type and prints a one-line summary (HR, SpO2/BP, temperature, weight/height/BMI). A GET to `http://172.16.22.191:8080/` returns the count received per device. Port 8080 must be open in the PC's firewall.

## OTA updates

From the config mode, the `ota` command connects to the stored WiFi and downloads `GW_OTA_URL` (GitHub release redirects are followed). The image's project name is checked before anything is written. Progress events go to the phone; on success the gateway restarts into the new firmware. A failed update leaves the running firmware untouched.

## Versioning

The firmware version is `MAJOR.MINOR.PATCH`, set in the top-level `CMakeLists.txt` (`GW_VERSION_MAJOR/MINOR/PATCH`). It is embedded in the binary and shown in the boot and status logs, the config `fw` field and the OTA log.

- **PATCH:** bug fix only.
- **MINOR:** new backwards-compatible feature (new command, new device).
- **MAJOR:** breaking change (JSON payload format, BLE config protocol).

Bump the version before every release build. Then upload `build/esp32s3_ble_gateway_opsolution.bin` as `firmware.bin` to the `ota_update` release.

## Project layout

```
main/
  main.c              boot flow, config-mode decision, status log
  Kconfig.projbuild   all gateway options
  include/            common.h, gateway.h, gw_config.h, gw_ota.h
  src/
    ble_central.c     scan, four slots, GATT handshake, watchdogs, status
    ecg_proto.c       ezFlex / ezBelt decoding and ECG JSON
    pc100_proto.c     PC-100 decoding and Nexus JSON
    tracky_proto.c    Tracky decoding and LS_TEMP JSON
    bf1h_proto.c      BF1H AiLink protocol and LS_BODYWH JSON
    uploader.c        PSRAM queue, HTTP/HTTPS POST, retries
    wifi.c            WiFi station, SNTP, host name
    gw_config.c       NVS settings: WiFi, slot URLs, slot MACs, mode
    config_mode.c     BLE config service for the phone app
    ota.c             HTTPS OTA
docs/BLE_CONFIG_PROTOCOL.md   phone app protocol
test/local_server.py          local HTTP receiver for testing
partitions.csv                16 MB partition table
sdkconfig.defaults*           build defaults
```
