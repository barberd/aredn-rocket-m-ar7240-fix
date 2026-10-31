# AREDN Rocket M AR7240 Fix

Fixes Ethernet, WiFi, and antenna selection on Ubiquiti Rocket M (XM) units with
the **AR7240** SoC.

## Problem

The Ubiquiti Rocket M (XM) shipped with two SoC variants:
- **AR7241** — works with stock AREDN ath79 images
- **AR7240** — completely broken: no Ethernet, no WiFi

Check yours with: `grep "system type" /proc/cpuinfo`

AREDN's `ubnt_rocket-m` image uses a device tree for the AR7241, which has two critical incompatibilities with the AR7240:

1. **Ethernet**: The built-in switch is on `mdio1` (AR7241) vs `mdio0` (AR7240). Wrong bus = switch never initializes = no link. ([openwrt/openwrt#6117](https://github.com/openwrt/openwrt/issues/6117))

2. **WiFi**: On the AR7240, the PCI wifi card enumerates with garbled device ID `168c:ff1c` because the EEPROM isn't loaded before bus scan. The old `ar71xx` target had a `pci-ath9k-fixup` for this; ath79 does not. ath9k never binds because `ff1c` isn't in its device table. ([openwrt/openwrt#6707](https://github.com/openwrt/openwrt/issues/6707))

3. **Antenna selection**: Even once WiFi works, AREDN can't identify the radio. `getBoardId()` in `files/usr/share/ucode/aredn/hardware.uc` identifies Ubiquiti radios by reading the PCI `subsystem_device`. On the AR7241 this returns `0xe1b2` (`Ubiquiti Rocket M2`), which resolves in `/etc/radios.json` to an `"antenna": "external"` entry and populates the antenna dropdown. On the AR7240 that register is left **unprogrammed and reads `0x0000`**, so the lookup falls through to `name: "Unknown"` — no radio name, no power profile, and an empty antenna list in the UI. (There is also a latent bug in the stock code: `subsystem_device` is read as `"0x0000\n"` with a trailing newline, so the existing `=== "0x0000"` comparisons never match and the `/dev/mtd7` fallback never runs.)

Both #1 and #2 are long-standing upstream OpenWrt bugs (2018/2019) that were never fixed for this hardware. #3 is an AREDN hardware-detection gap specific to the AR7240 revision.

## The Fix

Two patches are required.

### `patches/803-ubiquiti-rocket-m-ar7240.patch` (quilt patch, applied inside `openwrt/`)

1. Replaces the DTS with one based on `ar7240.dtsi` (switch on mdio0, correct PLL/MAC config, eth0 fixed-link at 100Mbps)
2. Changes the SOC identifier to `ar7240` in the image Makefile
3. Adds a mac80211 patch (`514-ath9k_add_ar7240_pci_id.patch`) that puts `168c:ff1c` in ath9k's PCI device table so the driver binds despite the garbled enumeration ID

### `patches/804-rocket-m-ar7240-antenna-id.patch` (applied at the AREDN repo root)

Adds an AR7240 fallback to `getBoardId()` in `files/usr/share/ucode/aredn/hardware.uc`: when the board is `ubnt,rocket-m` and no usable ID was read (empty or `0x0000`), it uses `0xe1b2` — the same ID the AR7241 sibling reports. That resolves to the `Ubiquiti Rocket M2` entry in `radios.json`, restoring the correct radio name, power profile (`maxpower 18`, `pwroffset 10`), and the full external 2.4 GHz antenna list. A `trim()` on the read value also fixes the trailing-newline comparison bug noted above.

> **Note on application scope:** `803` modifies files under `openwrt/`, so it goes through the quilt flow (`patches/series`). `804` modifies an AREDN-root file (`files/...`), which is symlinked into the build as-is and is **not** part of the quilt series — it is applied directly to the AREDN checkout with `patch -p1`. Do **not** add `804` to `patches/series`.

## Pre-built Firmware

**[Download from Releases](https://github.com/barberd/aredn-rocket-m-ar7240-fix/releases/latest)**

Built from **AREDN 4.26.7.0** (OpenWrt v25.12.5):

- `aredn-n3lp--93ad9ea9-ath79-generic-ubnt_rocket-m-squashfs-factory.bin` — TFTP/U-Boot flash
- `aredn-n3lp--93ad9ea9-ath79-generic-ubnt_rocket-m-squashfs-sysupgrade.bin` — sysupgrade

**⚠️ This image is AR7240-only. Do NOT flash on AR7241 hardware.**

## Flashing

1. Set your computer to `192.168.1.10/24`
2. Start TFTP server with the `factory.bin`
3. Hold reset while powering on the Rocket M
4. Wait for TFTP transfer and reboot
5. Device comes up at `192.168.1.1`

## Building From Source

```bash
git clone https://github.com/aredn/aredn.git
cd aredn
git checkout 4.26.7.0

# 803: quilt patch, applied inside openwrt/ via the series file
cp /path/to/803-ubiquiti-rocket-m-ar7240.patch patches/
echo "803-ubiquiti-rocket-m-ar7240.patch" >> patches/series

# 804: AREDN-root patch (files/...), applied directly — NOT added to series
patch -p1 < /path/to/804-rocket-m-ar7240-antenna-id.patch

# Edit config.mk: PRIVATE_BUILD_VERSION=<your callsign>
make
```

## References

- [openwrt/openwrt#6117](https://github.com/openwrt/openwrt/issues/6117) — AR7240 Ethernet broken on ath79
- [openwrt/openwrt#6707](https://github.com/openwrt/openwrt/issues/6707) — pci-ath9k-fixup never called on ath79
- [OpenWrt Wiki: Ubiquiti Airmax M](https://openwrt.org/toh/ubiquiti/airmaxm) — documents the AR7240 incompatibility
- `aredn/aredn: files/usr/share/ucode/aredn/hardware.uc` — `getBoardId()` radio identification logic
- `aredn/aredn: files/etc/radios.json` — radio ID → name/power/antenna map (`0xe1b2` = Ubiquiti Rocket M2)

## License

GPL-2.0-or-later, consistent with OpenWrt/AREDN.
