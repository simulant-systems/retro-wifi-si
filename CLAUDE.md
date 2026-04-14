# Retro WiFi SI Firmware

Firmware for the **Retro WiFi SI** WiFi modem by Simulant Systems (simulant.uk), for retro computers with RS-232 serial ports (Amiga, Atari ST, Amstrad CPC, C64 with adaptor, etc).

Based on **Zimodem 4.0.3** by Bo Zimmerman (https://github.com/bozimmerman/Zimodem).

## Hardware

- **MCU**: ESP8266EX (NodeMCU v2 / ESP-12E, 4MB flash)
- **Arduino FQBN**: `esp8266:esp8266:nodemcuv2`

## Build

```sh
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 zimodem --output-dir build/
cp build/zimodem.ino.bin bin/simulant-latest.bin
```

## Flash via USB

```sh
arduino-cli upload --fqbn esp8266:esp8266:nodemcuv2 --port /dev/ttyUSB0 --input-dir build/
```

## OTA release workflow

Releases go to two repos:

1. **`simulant-systems/retro-wifi-si`** (this repo) — source + `bin/simulant-latest.bin`
2. **`simulant-systems/zimodem-releases`** — `LATEST` (version string) + `simulant-latest.bin`

The device fetches via raw.githubusercontent.com:
- Version check: `raw.githubusercontent.com/simulant-systems/zimodem-releases/main/LATEST`
- Firmware: `raw.githubusercontent.com/simulant-systems/zimodem-releases/main/simulant-latest.bin`

```sh
cp build/zimodem.ino.bin bin/simulant-latest.bin
echo -n "X.Y.Z" > /tmp/zimodem-releases/LATEST
cp bin/simulant-latest.bin /tmp/zimodem-releases/simulant-latest.bin
cd /tmp/zimodem-releases && git add LATEST simulant-latest.bin && git commit -m "vX.Y.Z: description" && git push
```

Re-clone releases repo if needed: `git clone https://github.com/simulant-systems/zimodem-releases.git /tmp/zimodem-releases`

## Changes from upstream Zimodem 4.0.3

### 1. Suppress ESP8266 boot noise (`zimodem/zimodem.ino`)
```cpp
Serial.setDebugOutput(false);
```
Added at the top of the ESP8266 path in `setup()`. The ESP8266 SDK outputs debug info on UART before the sketch runs; without this it appears as garbage on the Amiga terminal.

### 2. Reset Amiga terminal charset on boot (`zimodem/zcommand.ino`)
```cpp
serial.printb(0x0f); // SI: cancel any spurious SO from ESP8266 bootloader noise
serial.prints(commandMode.EOLN);
```
Added immediately before the boot banner. The ESP8266 boot ROM output (which runs before `setup()` and cannot be suppressed) can include bytes the Amiga terminal interprets as SO (0x0E, shift-out), switching to the alternate character set and garbling lowercase. Sending SI (0x0F, shift-in) resets this.

### 3. OTA update — Simulant URLs + reliable download (`zimodem/wificlientnode.h`, `zimodem/proto_http.h`, `zimodem/proto_http.ino`, `zimodem/zcommand.ino`)

- `createWiFiClient()` gains an `sslRecvBuf` parameter and calls `setBufferSizes(sslRecvBuf, 512)` on the BearSSL client. The ESP8266 has ~27KB free heap after WiFi init; the default 16KB BearSSL buffer leaves no room for lwIP. Using 4096/512 for the version check and 2048/512 for the binary download fits within the available heap.
- `doWebGetStream()` gains `sslRecvBuf` and `rangeStart` parameters. The Range header allows resuming downloads from a byte offset.
- `doUpdateFirmware()` replaced: uses `raw.githubusercontent.com` (HTTPS, no redirects) instead of `www.zimmers.net`; downloads in 8192-byte chunks with TCP reconnects between each to prevent degradation from flash sector erases (~40ms with interrupts disabled); retries initial connection up to 5 times.

## Serial monitoring (non-destructive)

The modem appears as `/dev/ttyUSBx`. Monitor without resetting the device using O_NOCTTY:

```python
# /tmp/ota_monitor.py
import os, sys, termios, fcntl
dev = '/dev/ttyUSB0'
log = '/tmp/ota_log.txt'
fd = os.open(dev, os.O_RDWR | os.O_NOCTTY | os.O_NONBLOCK)
attrs = termios.tcgetattr(fd)
attrs[0] = 0; attrs[1] = 0
attrs[2] = termios.CS8 | termios.CREAD | termios.CLOCAL
attrs[3] = 0; attrs[4] = termios.B115200; attrs[5] = termios.B115200
attrs[6][termios.VMIN] = 0; attrs[6][termios.VTIME] = 0
termios.tcsetattr(fd, termios.TCSANOW, attrs)
flags = fcntl.fcntl(fd, fcntl.F_GETFL)
fcntl.fcntl(fd, fcntl.F_SETFL, flags & ~os.O_NONBLOCK)
with open(log, 'ab') as f:
    while True:
        try:
            data = os.read(fd, 256)
            if data: f.write(data); f.flush()
        except OSError: break
```

```sh
python3 /tmp/ota_monitor.py &
tail -f /tmp/ota_log.txt | strings
```

Kill monitor before flashing via USB, restart after.

## User's BBS

`amstrad.simulant.uk:464` — Synchronet BBS, connect with `ATDT"amstrad.simulant.uk:464"`
