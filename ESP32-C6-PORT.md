# ESP32-C6 (N16) port — local changes

This document describes the (currently **uncommitted, local-only**) changes made
to run this project on an **ESP32-C6-N16** board (16 MB flash, no PSRAM) in
addition to the originally supported ESP32-C3 (4 MB flash).

Status: not committed to git yet. Run `git status` / `git diff` to review before
committing.

## Why a C6 port makes sense here

The firmware's whole design — sorted 40-bit hashes binary-searched directly in
flash — never needed PSRAM and scales with flash size, not RAM. A bigger-flash
chip just means a bigger blocklist budget, nothing in the lookup/runtime logic
changes. The C6 is pin- and boot-button-compatible enough with the C3 that the
port only required a new PlatformIO environment, a new partition table sized
for 16 MB, and one `#if` guard fix.

## Files changed

### 1. `platformio.ini` — new `[env:c6]` build environment

Added alongside the existing `[env:c3]` (left untouched):

```ini
[env:c6]
board = esp32-c6-devkitc-1
board_build.partitions = partitions_c6_16mb.csv
board_upload.flash_size = 16MB
board_build.flash_size = 16MB
build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1
```

- `board = esp32-c6-devkitc-1` — closest stock board definition in the
  `espressif32` PlatformIO platform; works fine for generic C6-N16 boards
  (e.g. C6 SuperMini-style modules), same as how `esp32-c3-devkitm-1` is used
  for arbitrary C3 boards.
- `board_build.partitions` points at the new 16 MB partition table (see below)
  instead of the stock 4 MB `partitions.csv`.
- `board_upload.flash_size` / `board_build.flash_size` explicitly set to
  `16MB` — needed because the generic devkit board definition otherwise
  defaults to a smaller flash size and esptool would refuse/mis-flash a 16 MB
  chip.
- `build_flags` are copied unchanged from `[env:c3]`: the C6 also exposes a
  native USB-Serial-JTAG console, so the same `ARDUINO_USB_MODE` /
  `ARDUINO_USB_CDC_ON_BOOT` flags are required for `Serial` over USB to work
  (otherwise no serial output, same gotcha the C3 has).

Build/flash commands for this new environment:

```bash
pio run -e c6 -t upload
pio run -e c6 -t uploadfs
pio device monitor -e c6
```

(The `[env:esp32dev]` and `[env:c3]` environments are untouched; `default_envs`
in `[platformio]` is still `c3`, so you must pass `-e c6` explicitly.)

### 2. `partitions_c6_16mb.csv` — new file, 16 MB dual-OTA partition table

The stock `partitions.csv` is sized for a 4 MB C3 (two 1.3125 MB app slots +
1.3125 MB LittleFS). On a 16 MB chip that layout would waste ~12 MB. New table:

```csv
# Name,     Type, SubType, Offset,   Size
nvs,        data, nvs,     0x9000,   0x5000
otadata,    data, ota,     0xe000,   0x2000
app0,       app,  ota_0,   0x10000,  0x200000
app1,       app,  ota_1,   0x210000, 0x200000
spiffs,     data, spiffs,  0x410000, 0xBF0000
```

- Two **2 MB** app slots (`app0`/`app1`) for firmware OTA — much more headroom
  than the ~134 KB the C3 table has, future-proofing against firmware growth.
- The remaining **~12 MB** (`0xBF0000` ≈ 12,517,376 bytes) is given to the
  `spiffs` partition, mounted as LittleFS and used for the blocklist blob.
  At 5 bytes/hash entry, that's room for **~2.5 million domain hashes** — far
  beyond even the "ultimate" 537k-domain Hagezi list mentioned in the main
  README, which only fits the C3's single-app (no-OTA) table.
- `nvs` and `otadata` offsets/sizes are unchanged from the stock table (they're
  tiny and don't depend on total flash size).

No changes were needed in `tools/build_blocklist.py` — it just writes however
many hash entries fit the source lists; the generated `blocklist.bin` only
needs to be smaller than the `spiffs` partition size above.

### 3. `src/main.cpp` — BOOT button pin guard extended to C6

```diff
-#if CONFIG_IDF_TARGET_ESP32C3
-  const int BOOT_PIN = 9;     // C3 BOOT button
+#if CONFIG_IDF_TARGET_ESP32C3 || CONFIG_IDF_TARGET_ESP32C6
+  const int BOOT_PIN = 9;     // C3/C6 BOOT button
 #else
   const int BOOT_PIN = 0;     // classic ESP32 BOOT button (GPIO9 is a flash pin there)
 #endif
```

This is the "hold BOOT at power-on to wipe saved WiFi and force the setup
portal" check in `setup()`. The original code only recognized
`CONFIG_IDF_TARGET_ESP32C3` and otherwise fell back to GPIO0 (correct for the
classic ESP32, where GPIO9 is a flash strapping pin and can't be used as a
button). On the C6, like the C3, the BOOT button is on **GPIO9**, so without
this fix a C6 build would have silently used the wrong pin (GPIO0, which is
unconnected on most C6 devkits) and the "forget WiFi" gesture would never
trigger.

## What did NOT need to change

- **DNS sinkhole / hash-table lookup logic** — flash-size-agnostic by design,
  no PSRAM dependency on either chip.
- **Web dashboard, OTA endpoints, captive portal, mDNS** — all generic
  Arduino/ESP32 APIs, no C3-specific code paths.
- **`tools/build_blocklist.py`** — unchanged; just point it at bigger source
  lists if you want to use the extra flash headroom (e.g. the Hagezi
  "ultimate" list, ~537k domains, ~2.7 MB of hashes — comfortably inside the
  new ~12 MB `spiffs` partition).
- **`src/secrets.example.h` / `src/page.h`** — no board-specific content.

## Known caveats / things to verify on real hardware

- **PlatformIO platform version**: `esp32-c6-devkitc-1` requires a reasonably
  recent `espressif32` platform package (Arduino core 3.x / ESP-IDF 5.x
  support for the C6). If `pio run -e c6` fails with "unknown board", run
  `pio pkg update` to pull a current platform release.
- **Generic C6-N16 boards** (not the official DevKitC-1) may need a different
  `board =` entry if GPIO/flash pinout differs; `esp32-c6-devkitc-1` was
  chosen as the closest stock match, mirroring how `esp32-c3-devkitm-1` is
  used for arbitrary C3 boards in the existing `[env:c3]`.
- **BOOT pin assumption**: verified from the C6 DevKitC-1 schematic (GPIO9),
  but double-check if using a different C6-N16 board/module — some C6
  SuperMini-style boards wire BOOT to a different GPIO.
- **Flash size autodetection**: `board_upload.flash_size`/`board_build.flash_size`
  were hardcoded to `16MB` rather than relying on esptool's `--flash_size detect`,
  to make the partition table assumption explicit and fail loudly instead of
  silently truncating on a smaller chip.
- Not yet flashed/tested on physical ESP32-C6-N16 hardware as of writing this
  document — only the config/code changes have been made and compiled logic
  reviewed.

## Reviewing / committing these changes

Changes are currently local-only. To review:

```bash
git status
git diff -- platformio.ini src/main.cpp
git status --porcelain  # partitions_c6_16mb.csv and this file show as untracked
```

To commit (suggested, not yet done):

```bash
git add platformio.ini partitions_c6_16mb.csv src/main.cpp ESP32-C6-PORT.md
git commit -m "Add ESP32-C6 (16MB) build environment and partition table"
```
