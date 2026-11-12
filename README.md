<p align="center">
  <img alt='LIVI' src='docs/media/banner.png' width="1200" />
</p>

# LIVI - Linux In-Vehicle Infotainment (UDP Navigation Telemetry Fork)

This repository is a fork of [f-io/LIVI](https://github.com/f-io/LIVI) with a lightweight local UDP telemetry bridge added to the Android Auto navigation stack.

---

## What's Modified in This Fork

* **Android Auto Navigation UDP Forwarder (`src/main/services/projection/driver/aa/stack/channels/NavigationChannel.ts`):**
  * Added a non-blocking Node.js `dgram` (`udp4`) socket streaming live Android Auto navigation events as JSON payloads to `127.0.0.1:5005`.
  * Forwarded events:
    * `nav-turn` (turn direction, side, angle, street name — stripping the raw PNG `image` buffer to keep UDP datagrams lightweight)
    * `nav-distance` (remaining distance and display unit)
    * `nav-state` (navigation active state and destination address)
    * `nav-position` (time to arrival and step distance)
  * Designed to feed external vehicle integration daemons (such as a Python `SocketCAN` bridge forwarding turn-by-turn prompts to a Fiat Stilo 50 kbps B-CAN instrument cluster).

---

## Base Project Overview

LIVI is an open-source **Apple CarPlay and Android Auto head unit**.

It is a standalone cross-platform head unit with a native, zero-copy GStreamer video pipeline and hardware-accelerated decoding on Linux (including the Raspberry Pi 4 and 5) and macOS, low-latency audio, multitouch + D-Pad navigation, and support for very small embedded/OEM displays.

## Native Connectivity

- **Apple CarPlay** (wired & wireless) on Linux and Mac — requires [MFi Authentication](#mfi-authentication)
- **Android Auto** (wired) on all platforms
- **Android Auto** (wireless) on Linux

## Native Apple CarPlay

LIVI implements the CarPlay accessory side natively, wired and wireless, on Linux and on macOS. Wireless sessions run over LIVI's own Wi-Fi access point with Bluetooth pairing, wired sessions run directly over the USB cable (no OTG required).

- main + instrument cluster video (H.264/H.265, hardware decoded, zero-copy)
- audio playback, phone calls, microphone uplink
- now-playing metadata incl. album art, turn-by-turn navigation data
- touch, knob/D-Pad and hard-key input
- day/night mode and GPS forwarding to the phone
- multi-session with live switching between connected phones

Wireless CarPlay requires a Bluetooth adapter and a Wi-Fi interface dedicated to the access point. Both can come from a [LIVI Link](LIVI-LINK.md), which brings its own radios. Wired CarPlay works on any USB port.

## MFi Authentication

CarPlay requires the accessory to authenticate against the phone. The usual way is an Apple **MFi authentication coprocessor**, a hardware chip that cannot be emulated and that LIVI neither ships nor bypasses. Two routes:

- **On the board** — a coprocessor wired to the I²C bus
- **Over the network** — a [LIVI Link](LIVI-LINK.md)

Both routes work on Linux. A Mac has no I²C bus to put a coprocessor on, so there the LIVI Link is the only one.

For a chip on the board, configuration (`config.json`):

| Key                   | Default | Description                          |
| --------------------- | ------- | ------------------------------------ |
| `carPlayMfiI2cBus`    | `2`     | I²C bus number the coprocessor is on |
| `carPlayMfiPowerGpio` | `21`    | GPIO that powers the coprocessor     |

Without one of these, native CarPlay is unavailable. All Android Auto paths work regardless.

## Wireless

Wireless sessions do not need a router. LIVI brings up its own Wi-Fi access point and the phone joins that. Bluetooth carries the pairing and the handover, the session itself then runs over Wi-Fi.

Wireless CarPlay and wireless Android Auto are enabled separately, so a head unit can offer one, both, or neither. With Auto Connect on, a phone that has been paired before is picked up again on its own. The car name is what the phone shows when it lists nearby vehicles.

The Wi-Fi page sets the band, password, channel and country for the access point and picks the Wi-Fi interface. The Bluetooth adapter is picked next to it, under Connections.

Dedicated Interface reserves the Wi-Fi adapter for the access point and brings it up during boot, out of NetworkManager's hands. Without it the access point is started on demand and the interface is handed back afterwards, which keeps it available for normal networking but costs a moment on the first connection.

Configure under Settings → General → Connections.

<p align="center">
  <img src="docs/media/connections.png" alt="Connection settings" width="42%" align="top" />
  &emsp;
  <img src="docs/media/wifi.png" alt="Wi-Fi access point settings" width="42%" align="top" />
</p>

## Multi-Session

Several phones can be connected at the same time. All of them are tracked live: every session keeps its state, its battery and signal readings, its media metadata and its turn-by-turn navigation up to date in the background, not just the one on screen.

Switching is a handover, not a reconnect. The drivers stay armed and the video and audio paths stay open, so bringing another phone forward takes milliseconds instead of multiple seconds.

Manage under Settings → Devices.

<p align="center">
  <img src="docs/media/devices.png" alt="Connected devices" width="70%" />
</p>

## Display Calibration

Gamma, contrast, and the red, green and blue channels are adjustable independently. The correction is applied by the compositor as a single pass over the finished frame, so it covers everything on screen: the LIVI interface, the dashboards, and the projected CarPlay or Android Auto video alike.

Configure under Settings → Appearance → Contrast / Gamma and Settings → Appearance → Color.

<p align="center">
  <img src="docs/media/contrast_gamma.png" alt="Contrast and gamma calibration" width="42%" align="top" />
  &emsp;
  <img src="docs/media/color.png" alt="Colour channel calibration" width="42%" align="top" />
</p>

## UDP Navigation Stream Format

When Android Auto navigation is active, LIVI emits JSON datagrams to `127.0.0.1:5005`:

```json
{
  "event": "nav-turn",
  "data": {
    "turnSide": 1,
    "turnAngle": 90,
    "roadName": "Ady Endre u."
  }
}
```

Any local listener bound to `UDP 127.0.0.1:5005` can consume these events in real time without modifying LIVI's core IPC layer.

## Installation

> [!IMPORTANT]
> LIVI requires **OpenGL ES 3.x**.

One script installs LIVI on both desktop and headless hosts, auto-detecting which (override with `--desktop` / `--headless`). It uses apt or dnf, so beyond Raspberry Pi OS it also runs on Debian/Ubuntu, Fedora and likely other distros, on arm64 and x86_64.

```bash
curl -fL -o install.sh https://raw.githubusercontent.com/f-io/LIVI/main/scripts/install/install.sh
chmod +x install.sh
./install.sh
```

_Tested on Raspberry Pi OS, Ubuntu 26.04 and Fedora 44._

## Build Environment

### System Requirements (Debian / Ubuntu / Raspberry Pi OS)

```bash
sudo apt-get update
sudo apt-get install -y git build-essential \
  pkg-config cmake \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
  libwayland-dev libxkbcommon-dev \
  fuse3 bluez libspa-0.2-bluetooth hostapd dnsmasq-base iw rfkill avahi-daemon avahi-utils \
  pulseaudio-utils
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo corepack enable
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

### Clone & Build

```bash
# Git clone
git clone --branch main --single-branch https://github.com/ItzNightBlade/LIVI.git \
  && cd LIVI

# Install dependencies from lockfile
pnpm run install:ci

# --- Build targets ---

# Linux (AppImage)
pnpm run build:linux:arm64         # ARM (Raspberry Pi 4/5)
pnpm run build:linux:x64           # X86_64

# macOS (dmg)
pnpm run build:mac:arm64           # Apple Silicon
pnpm run build:mac:x64             # Intel
```

## Debugging

Diagnostic environment flags and where to find the logs are documented in [DEBUGGING.md](DEBUGGING.md).

## Credits

See [CREDITS](CREDITS.md) for acknowledgements and prior art.

## Disclaimer

_Apple and CarPlay are trademarks of Apple Inc. Android and Android Auto are trademarks of Google LLC. This project is not affiliated with or endorsed by Apple or Google. All product names, logos, and brands are the property of their respective owners._

## License

LIVI is free software, licensed under the **GNU General Public License v3.0 or later** (`GPL-3.0-or-later`). See [LICENSE](LICENSE) for the full text.

Copyright (C) 2025 Lasse Heitgres  
UDP Navigation Telemetry Bridge Copyright (C) 2026 Czeilinger Dominik József (ItzNightBlade)
