# Corne Wireless ZMK Config

Corne split keyboard using a Raytac MDBT50Q-RX USB dongle as the central. Both halves connect wirelessly to the dongle, which connects to the host via USB HID.

## Hardware

| Role | Board | Shield |
|---|---|---|
| Central (dongle) | Raytac MDBT50Q-RX | `corne_dongle` (custom, in this repo) |
| Left peripheral | nice!nano v2 | `corne_left` + nice!view |
| Right peripheral | nice!nano v2 | `corne_right` + nice!view |

## Firmware artifacts

GitHub Actions builds these artifacts from `build.yaml`:

| Artifact | Flash to |
|---|---|
| `corne_dongle.uf2` | Dongle |
| `corne_left_peripheral.uf2` | Left half |
| `corne_right_peripheral.uf2` | Right half |
| `raytac_settings_reset.uf2` | Dongle (bond reset only) |
| `nice_nano_settings_reset.uf2` | Either half (bond reset only) |
| `corne_dongle_debug.uf2` | Dongle (USB logging enabled for diagnostics) |

## Flashing

### Entering bootloader mode

**Dongle (Raytac MDBT50Q-RX):** Hold the small button while plugging in over USB. The dongle mounts as a UF2 mass-storage drive. Double-clicking the reset button does NOT work on this board.

**nice!nano v2:** Double-click the reset button. The board mounts as a UF2 mass-storage drive.

### First-time setup or after firmware changes that affect bonding

All three devices must have their stored bond data cleared before they will pair with each other. Skipping this step results in the halves showing "no connection" indefinitely.

1. Flash `raytac_settings_reset.uf2` to the dongle
2. Flash `nice_nano_settings_reset.uf2` to the left half
3. Flash `nice_nano_settings_reset.uf2` to the right half
4. Flash `corne_dongle.uf2` to the dongle
5. Flash `corne_left_peripheral.uf2` to the left half
6. Flash `corne_right_peripheral.uf2` to the right half
7. Plug the dongle in first, then power both halves within a few seconds

The halves advertise openly after a settings reset; the dongle scans and bonds automatically.

### Keymap-only changes

Reflash only `corne_dongle.uf2`. No settings reset needed.

## Architecture notes

### Why `corne_dongle.conf` explicitly enables BLE and USB

The Raytac v0.3 board defconfig sets `CONFIG_ZMK_BLE=y` and `CONFIG_ZMK_USB=y`, but these do not survive the ZMK v0.3 Kconfig merge — they are silently discarded, leaving the dongle with no Bluetooth stack. The result is a ~50KB USB-only firmware that cannot scan for or connect to the halves.

The explicit assignments in `config/corne_dongle.conf` override this:

```
CONFIG_ZMK_USB=y
CONFIG_ZMK_BLE=y
```

**Do not remove these lines.** Without them, the dongle will appear to flash successfully and show as a USB HID keyboard but will never connect to either half.

### Why halves use cmake-args instead of conf files

The Corne left shield (`SHIELD_CORNE_LEFT`) sets `ZMK_SPLIT_ROLE_CENTRAL=y` by default. To override this so the left half becomes a peripheral under the dongle, `build.yaml` passes:

```
cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
```

This is the approach documented in the [ZMK dongle guide](https://zmk.dev/docs/hardware-integration/dongle).

### Dongle shield location

The `corne_dongle` shield lives in `boards/shields/corne/` inside this config repo. It is not part of the upstream ZMK Corne shield. The `zephyr/module.yml` at the repo root tells the ZMK build system to treat this repo as a Zephyr module so the shield is discovered automatically.

The dongle overlay includes the upstream `corne.dtsi` transform/layout definitions by re-declaring the foostan layout includes and matrix transforms directly — a relative `#include "corne.dtsi"` cannot be used because the upstream file lives in a different directory that is not on the preprocessor include path.

### ZMK version

Pinned to ZMK v0.3 and `zmk-component-raytac-dongle` v0.3 via `config/west.yml`. Both must stay in sync.
