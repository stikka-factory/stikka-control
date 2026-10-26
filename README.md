# Stikka-Controller

A small (42 × 23 mm, 2-layer) carrier board for an **ESP32-S3-WROOM-1** module
that turns a label printer into a network printer for the Stikka sticker
factory. The board runs the [stikka-esp32](https://github.com/stikka-factory/stikka-esp32)
firmware, which receives print jobs over MQTT from the
[stikka-mqtt](https://github.com/stikka-factory/stikka-mqtt) web app and
forwards them to the printer, normally over the printer's own USB port with
the ESP32-S3 acting as a USB host.

```
Browser (stikka-mqtt)  ──MQTT──▶  broker  ──MQTT──▶  Stikka-Controller  ──USB──▶  printer
render label, publish                                 (stikka-esp32 fw)    J7
to /<printer>/command/                                status on /<printer>/status/
```

## Repository contents

| Path | What it is |
|---|---|
| `Stikka-Controller.kicad_pro` / `.kicad_sch` / `.kicad_pcb` | KiCad project, schematic and layout |
| `bom/ibom.html` | Interactive BOM (open in a browser for assembly) |
| `production/Stikka-Controller.zip` | Gerbers + drill files for fabrication |
| `production/netlist.ipc` | IPC-D-356 netlist for fab electrical test |

## Main parts

| Ref | Part | Function |
|---|---|---|
| U1 | ESP32-S3-WROOM-1 | MCU, Wi-Fi. Use the **N16R8** variant (16 MB flash, 8 MB octal PSRAM): the firmware is built for it and needs the PSRAM. |
| U5 | RECOM R-78E5.0-1.0 footprint (schematic value says RBT10W24S05) | Switching regulator, VIN → +5 V. Feeds the printer's USB VBUS, the status LED and the LDO. |
| U2 | AP2127N-3.3 (SOT-23) | LDO, +5 V → +3.3 V for the ESP32. |
| Q1, Q2 + R1, R2 (10k) | SS8050 (SOT-23) | Classic DTR/RTS auto-program circuit, so a USB-UART adapter can reset the ESP32 and put it in download mode. |
| R3 (10k) + C1 (1µ) | | EN pull-up and power-on reset delay. |
| C11–C16 | 1µ / 4µ7 / 10µ, 0603 | Rail decoupling. |
| J7 | Grove-style 4-pin, 2.0 mm, SMD right angle | USB host port to the printer. |
| J4 | 1×5 pin header, 2.54 mm | Programming / debug UART. |
| J6 | 1×3, 2.0 mm (Samtec UMPT footprint) | External WS2812 / NeoPixel status LED. |
| J1, J2, J3 | 2-pad SMD (uses the `L_APV_ANR252010` footprint) | Reset pads, boot pads, power input. |

## Connectors and pinout

### J3: power input

| Pin | Net |
|---|---|
| 1 | VIN |
| 2 | GND |

VIN goes straight into U5. The usable input range depends on the regulator
actually fitted: an R-78E5.0-1.0 takes 7–28 V and gives up to 1 A at 5 V.
That 1 A is shared between the ESP32 and the printer's VBUS. There is no
reverse-polarity protection.

### J7: USB host to the printer

| Pin | Net | ESP32-S3 |
|---|---|---|
| 1 | GND | |
| 2 | +5V (VBUS) | |
| 3 | USB_D- | GPIO19 |
| 4 | USB_D+ | GPIO20 |
| SS1, SS2 | GND (mounting tabs) | |

This is a USB 1.1 full-speed host port on the ESP32-S3's native USB-OTG pins.
It uses a Grove housing, but it is **not I²C**, so don't plug Grove modules
into it. Wire it to a USB-A socket or straight to a USB-B plug for the
printer: GND → pin 4 (black), VBUS → pin 1 (red), D- → pin 2 (white),
D+ → pin 3 (green).

### J4: programming / debug UART

| Pin | Net | ESP32-S3 |
|---|---|---|
| 1 | TXD | GPIO43 (U0TXD), output from the board |
| 2 | RXD | GPIO44 (U0RXD), input to the board |
| 3 | DTR | Q2 emitter, Q1 base via R1 |
| 4 | RTS | Q1 emitter, Q2 base via R2 |
| 5 | GND | |

Connect a 3.3 V USB-UART adapter that exposes DTR and RTS (CP2102, CH340 and
similar): adapter RX → J4 TXD, adapter TX → J4 RXD. With DTR/RTS connected,
esptool, PlatformIO and the web flasher in stikka-mqtt reset the board into
download mode on their own. J4 has no power pin, so power the board through
J3 while flashing.

The same port carries the firmware's debug log at 115200 baud (`pio device
monitor`).

### J6: status LED

| Pin | Net | ESP32-S3 |
|---|---|---|
| 1 | +5V | |
| 2 | LED (data) | GPIO48 |
| 3 | GND | |

For a single WS2812 / NeoPixel (GRB). The LED runs from 5 V while the data
line is 3.3 V, which most WS2812Bs accept but which is below spec
(V<sub>IH</sub> = 0.7 × VDD ≈ 3.5 V). If an LED misbehaves, use a WS2812
variant that is happy with 3.3 V logic, or a level shifter.

### J1, J2: reset and boot pads

| Ref | Pin 1 | Pin 2 | Use |
|---|---|---|---|
| J1 | RST (EN) | GND | Short to reset the ESP32 |
| J2 | BOOT (GPIO0) | GND | Hold shorted while resetting to enter the ROM bootloader |

You only need these if your USB-UART adapter has no DTR/RTS.

### ESP32-S3 GPIO use

| GPIO | Net | Used for |
|---|---|---|
| EN | RST | Reset (R3/C1, Q1, J1) |
| 0 | BOOT | Strapping / download mode (Q2, J2) |
| 19 | USB_D- | USB host to printer (J7) |
| 20 | USB_D+ | USB host to printer (J7) |
| 43 | TXD | UART0 TX (J4) |
| 44 | RXD | UART0 RX (J4) |
| 48 | LED | NeoPixel data (J6) |

All other GPIOs are unconnected and not broken out.

## Firmware: stikka-esp32

Use the single `esp32-s3-devkitc-1` env from stikka-esp32's `platformio.ini`.
It targets the same N16R8 module (`qio_opi` memory type, `BOARD_HAS_PSRAM`,
16 MB partitions), so it runs on this board as is. You can flash it in
either of two ways:

- `pio run -e esp32-s3-devkitc-1 -t upload` from a stikka-esp32 checkout,
  with the USB-UART adapter on J4.
- The in-browser ESP32 flasher in the stikka-mqtt web app, over the same
  adapter.

On first boot the firmware opens a setup Wi-Fi AP, and its web wizard asks
for the printer protocol and connection method. On this board, choose:

| Method | Works on this board? | Notes |
|---|---|---|
| `usb_host` | **Yes**, the intended use | J7. The board supplies VBUS from its own 5 V rail. Covers ZPL, Brother QL and Seiko printers on USB. |
| `network` | Yes | No board hardware involved; the ESP32 talks to a ZPL printer over TCP. |
| `serial` | **No** | No UART for the printer is broken out (see below). |

### Firmware/board pin check

Checked against stikka-esp32 `3e263b6`:

| Function | Firmware | Board | Status |
|---|---|---|---|
| Status LED | NeoPixel on GPIO48, GRB (`DEFAULT_LED_*` in `platformio.ini`) | J6 pin 2 = GPIO48 | ✅ matches |
| Debug log | `Serial` = UART0, 115200 (`ARDUINO_USB_CDC_ON_BOOT=0`, `logging.cpp`) | J4 TXD/RXD = GPIO43/44 | ✅ matches |
| USB host | Native USB-OTG (`usb_host_target.cpp`) | J7 D-/D+ = GPIO19/20, VBUS = +5V | ✅ matches |
| Serial printer | UART2 on `printerUartTxPin=32` / `printerUartRxPin=33` (`config.h`) | Not broken out | ⚠️ GPIO32/33 aren't pins on the WROOM-1 module. On the N16R8 they're used internally by the flash and octal PSRAM, so this default doesn't work on this board, or on any WROOM-1 board. |

## Using it with stikka-mqtt

Once the board is on Wi-Fi and connected to the same MQTT broker as the
stikka-mqtt web app, it publishes a retained status snapshot on
`/<printerName>/status/`. The web app picks that up on its `/+/status/#`
subscription and lists the printer. Print jobs come in on
`/<printerName>/command/`, and the board's status LED shows the connection
state:

| Colour | Meaning |
|---|---|
| Green | Wi-Fi and MQTT connected |
| Yellow | Wi-Fi only |
| Red | No Wi-Fi (setup AP mode) |
| Purple / cyan flash | MQTT message received / sent |

stikka-mqtt's README documents the message format.

## Known issues / open points

- **U5 value vs footprint:** the schematic value is `RBT10W24S05`, but the
  symbol and footprint are RECOM R-78E5.0-1.0. Check which part is actually
  fitted before choosing a supply voltage.
- **J1–J3 footprint:** these use an inductor footprint (`L_APV_ANR252010`) as
  plain 2-pad SMD pads. That's fine electrically, but it's easy to miss in
  the BOM.
- **5 V budget:** with a 1 A regulator, the printer's USB port only gets what
  the ESP32 leaves over (~0.5–0.7 A while Wi-Fi is transmitting). That's
  enough for printers powered from their own supply, but not for
  bus-powered ones.
- **No serial-printer header:** if `serial` support is wanted on this board,
  a later revision would need to break out a free GPIO pair (e.g. GPIO17/18,
  the U1TXD/U1RXD defaults), and the firmware defaults would need to change
  to match.
