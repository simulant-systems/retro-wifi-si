# Retro WiFi SI Firmware — Build Notes

Firmware for the **Retro WiFi SI** WiFi modem by Simulant Systems (simulant.uk).
Based on **Zimodem 4.0.3** by Bo Zimmerman (https://github.com/bozimmerman/Zimodem).

## Hardware

- **MCU**: ESP8266EX (NodeMCU v2 / ESP-12E, 4MB flash)
- **Arduino FQBN**: `esp8266:esp8266:nodemcuv2`

## Build

```sh
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 zimodem --output-dir build/
```

## Flash via USB

```sh
arduino-cli upload --fqbn esp8266:esp8266:nodemcuv2 --port /dev/ttyUSB0 --input-dir build/
```

## OTA release workflow

Releases are served from this repo via raw.githubusercontent.com:
- Version check: `raw.githubusercontent.com/simulant-systems/retro-wifi-si/main/LATEST`
- Firmware binary: `raw.githubusercontent.com/simulant-systems/retro-wifi-si/main/simulant-latest.bin`

To publish a new release:

```sh
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 zimodem --output-dir build/
cp build/zimodem.ino.bin simulant-latest.bin
echo -n "X.Y.Z" > LATEST
git add LATEST simulant-latest.bin
git commit -m "vX.Y.Z: description"
git push
```

## Changes from upstream Zimodem 4.0.3

### 1. Suppress ESP8266 boot noise (`zimodem/zimodem.ino`)

```cpp
Serial.setDebugOutput(false);
```

Added in `setup()`. The ESP8266 SDK outputs debug info on UART before the sketch runs,
which appears as garbage on the connected terminal.

### 2. Reset terminal charset on boot (`zimodem/zcommand.ino`)

```cpp
serial.printb(0x0f); // SI: cancel any spurious SO from ESP8266 bootloader noise
```

Added before the boot banner. The ESP8266 boot ROM can output bytes that some terminals
interpret as SO (shift-out), switching to an alternate character set and garbling lowercase.
Sending SI (shift-in) resets this before the banner is displayed.

### 3. OTA update — Simulant URLs + reliable download (`zimodem/wificlientnode.h`, `zimodem/zcommand.ino`)

- `createWiFiClient()`: ESP8266 SSL enabled with `setBufferSizes(4096, 512)` to fit within
  available heap (~27KB after WiFi init).
- `doUpdateFirmware()`: replaced with Simulant URLs on `raw.githubusercontent.com` (HTTPS,
  no redirects). Downloads in 8192-byte chunks with TCP reconnects between each, to prevent
  degradation from flash sector erases. All OTA logic is self-contained.

### 4. Branding

- Boot banner updated to identify as Retro WiFi SI Modem with attribution to Bo Zimmerman's Zimodem.
- Spelling: INITIALISED (British English).
