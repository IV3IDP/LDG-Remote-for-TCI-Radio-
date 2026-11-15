# LDG Remote for TCI Radio

Remote control of **LDG AT-1000ProII** and **AT-600ProII** automatic antenna tuners from a PC, a phone or a tablet, with **TCI** integration for SDR radios (e.g. Hermes Lite 2) to generate the tuning carrier.

*by IV3IDP* · [Italiano](README.it.md)

> **Status:** working prototype, developed and tested with an **AT-600ProII**. The AT-1000ProII uses the same protocol but has not been tested or calibrated yet. See [Known limitations](#known-limitations).

<!-- Add a screenshot: docs/screenshot.png -->

## Features

- **Two interfaces, one core**
  - Native Windows window (WPF) with a dark theme.
  - Web interface in the browser, usable from other devices on the local network.
- **Tuner control** for both models: antenna toggle (A1/A2), Bypass, Auto, Manual, Memory Tune and Full Tune. Two tuners can be connected at the same time, one serial port each.
- **Live measurements:** frequency (MHz), forward power, reflected power, SWR, active antenna, mode and last tune result.
- **LED bar graphs** for power (100 W / 1000 W scale) and SWR, with green / amber / red zones.
- **Tune with a carrier from the radio via TCI:** the program starts the carrier (`tune:0,true;`), runs the tune and always switches the carrier off at the end, even on errors. Automatic reconnection to the TCI server.
- **Network access with a password** (HTTP Basic). From `localhost` no password is needed; from other devices a password is required, otherwise external requests are blocked.
- **Raw capture log** of every byte sent and received (`capture-*.log`), useful to debug and to extend the protocol.
- Splash screen, self-installing package (Inno Setup), per-user configuration.

## How it works

The tuners expose a 4-pin mini-DIN **meter port** (originally for the LDG M-1000 meter): TTL serial, 38400 baud, 8N1. The program talks to it through a USB-TTL adapter, decodes the telemetry stream and sends single-character commands.

```
Radio (TCI) <── WebSocket ──┐
                            ├── LdgCore ── USB-TTL ── AT-600ProII / AT-1000ProII
Native window / Web page ───┘
```

## Requirements

- Windows 64-bit (the native window is Windows only; the web version also runs on Linux/macOS from source).
- A **5 V TTL USB-serial adapter** (FTDI, CH340, CP210x...) for each tuner, and a cable with the 4-pin mini-DIN plug.
- A radio or SDR program with a **TCI server** (optional: needed for tuning with a carrier).
- **.NET 8**, unless you use the self-contained installer.

## Hardware connection

| Tuner pin | Signal | Connect to |
|---|---|---|
| 1 | +12 V (power for the meter) | **do not connect** |
| 2 | GND | adapter GND |
| 3 | METERRX (tuner input) | adapter **TXD** |
| 4 | METERTX (tuner output) | adapter **RXD** |

TX and RX are crossed (names are from the tuner's point of view). Pin numbering follows the standard 4-pin mini-DIN; verify with a multimeter before wiring: with the tuner on, pin 1 must read about +12 V against pin 2. Never connect pin 1 to the adapter. A normal S-Video cable is not suitable because it ties some conductors together.

## Installation

### Installer

Build one of the installers (requires the .NET 8 SDK and [Inno Setup 6](https://jrsoftware.org/isinfo.php)):

| Script | Output | Runtime |
|---|---|---|
| `build-installer.bat` | `installer\Output\LdgSuite-Setup.exe` | included (~80-120 MB) |
| `build-installer-light.bat` | `installer\Output\LdgSuite-Setup-Light.exe` | requires .NET Desktop Runtime 8 and ASP.NET Core Runtime 8 (x64) |

The installer creates Start menu shortcuts, an optional desktop icon and an optional firewall rule for the web version. Configuration and logs go to `%LOCALAPPDATA%\LdgRemote`, shared by the native and the web version.

### From source

```
create-solution.bat     # once: creates LdgSuite.sln
run-native.bat          # native window
run-web.bat             # web version, opens http://localhost:5000
```

## Configuration

`appsettings.json` (installed: `%LOCALAPPDATA%\LdgRemote\appsettings.json`; from source: next to the executable):

```json
{
  "Urls": "http://localhost:5000",
  "Auth": { "User": "ldg", "Password": "" },
  "OpenBrowser": true,
  "CaptureRaw": true,
  "Tci": { "Url": "ws://localhost:50001", "Trx": 0 },
  "Tuners": [
    { "Id": "1000", "Name": "AT-1000ProII", "Port": "COM5",
      "Fwd": { "Scale": 1.0 }, "Ref": { "Scale": 1.0 } },
    { "Id": "600", "Name": "AT-600ProII", "Port": "COM6", "BandConstant": 16415,
      "Fwd": { "Scale": 29.0, "Voltage": true }, "Ref": { "Scale": 29.0, "Voltage": true } }
  ]
}
```

- **Port:** exact name (`COM3`, `/dev/ttyUSB0`) or part of the name / adapter serial number (e.g. `FTDI0F83`). At start the program lists the serial ports it finds. With two identical adapters on Windows, assign fixed COM numbers in Device Manager.
- **Calibration (`Fwd`, `Ref`):** `Scale` and `Voltage` (watts = (raw / Scale)² when `true`, raw × Scale when `false`), or `Points`, a list of `[raw, watts]` pairs with piecewise-linear interpolation, to refine later.
- **BandConstant:** MHz = `BandConstant` / band-field value (see below).
- **Network access:** set `"Urls": "http://0.0.0.0:5000"` and a non-empty `Auth.Password`, then allow the port in the firewall (`open-firewall.bat` or the installer option). The page is then reachable at `http://<PC address>:5000`. If the port is busy, the next free one is used and printed in the console.

> The connection is plain HTTP and the password travels unencrypted: fine on a trusted LAN, but **do not expose the port to the Internet**. Use a VPN (WireGuard, Tailscale...) for remote use. Anyone who logs in can operate the tuners and key the radio.

## Protocol notes

Observed on an AT-600ProII with a 5-minute capture; contributions for other models are welcome.

**Commands** (single byte, one response byte):

| Action | Command | Response | Notes |
|---|---|---|---|
| Toggle antenna | `A` | `1` / `2` | ~90 ms |
| Memory tune | `T` | `T` SWR < 1.5 · `M` SWR 1.5–3.0 · `F` failed | `F` after ~2 s if there is no RF |
| Full tune | `F` | same | same |
| Bypass | `P` | `P` | ~0.75 s |
| Auto mode | `C` | `A` | documented, not yet captured |
| Manual mode | `M` | `M` | documented, not yet captured |

Mode and framing: `X` = control mode, `S` = meter mode, a space wakes the tuner. The program sends `X`, a space, the command and then `S`.

**Telemetry packet** (8 bytes, big-endian):

| Bytes | Field |
|---|---|
| 0–1 | forward power (raw) |
| 2–3 | reflected power (raw) |
| 4–5 | band / frequency counter |
| 6–7 | terminator `0x3B 0x3B` (`;;`) |

- The stream is **continuous (~130 packets/s) while RF is present** and only short bursts (~0.2 s) after a command otherwise.
- The **response byte is always inserted between two packets**, right after a `;;`; the parser separates the two.
- **Band field:** inversely proportional to the frequency. MHz ≈ 16415 / value (2292 → 7.166 MHz, 1146 → 14.317 MHz, resolution ~3 kHz at 7 MHz and ~12 kHz at 14 MHz). `5958` means "no measurement" and `65535` "invalid".
- **Power scale:** with 4 W the forward value is about 54–63 depending on the band. The default assumes a value proportional to the detector voltage (watts = (raw / 29)²) and SWR = (1 + ρ) / (1 − ρ) with ρ = reflected / forward. This is a hypothesis to be confirmed with carriers at other powers.

## Project structure

```
LdgCore/     shared library: serial protocol, stream parser, TCI client, calibration, TunerService
LdgRemote/   ASP.NET Core web app (WebSocket + page in wwwroot)
LdgNative/   native WPF window
installer/   Inno Setup script
```

## Known limitations

- Not yet available: the C/L relay LEDs and the C Up/Dn, L Up/Dn, Func and Power buttons (commands unknown).
- A Full Tune fails if the radio does not keep the carrier on for the whole tune.
- Power calibration is provisional; AT-1000ProII values are shown raw and its band constant is unverified.
- The frequency formula was verified on two bands (7 and 14 MHz) of the AT-600ProII.
- The native window is Windows only.

## Contributing

The most useful contribution is a **capture log** (`capture-*.log`, only timestamps and serial bytes) from your tuner, together with what you did: carrier frequency and power, commands pressed on the front panel, model. With it the protocol, the calibration and the missing commands (C/L relays) can be extended. Issues and pull requests are welcome.

## Acknowledgements

- LDG Electronics for the published M-1000 meter protocol document.
- [LDGControl](https://github.com/frazierjason/LDGControl) for the analysis of the meter protocol.
- LDG-Control by GW3JVB, which inspired the layout of the interface.

## Disclaimer

This is an independent project, **not affiliated with LDG Electronics**. LDG and the AT-1000ProII / AT-600ProII names are trademarks of their respective owners. The software can key a radio transmitter: use it at your own risk and in accordance with your license conditions.

## License
