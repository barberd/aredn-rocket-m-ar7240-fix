# AREDN Rocket M AR7240 Ethernet Fix

## Problem

The Ubiquiti Rocket M (XM) exists in two hardware variants:
- **AR7241** — works fine with AREDN 4.26.x on ath79
- **AR7240** — Ethernet is completely non-functional (NO-CARRIER, no link)

AREDN's `ubnt_rocket-m` image uses a device tree (DTS) based on `ar7241.dtsi`, which places the built-in Ethernet switch on **mdio1**. On the AR7240, the switch is on **mdio0**. This mismatch causes the switch driver to fail silently — it reads PHY UID as `0x00000000`, the Generic PHY driver is used, and no link is ever established.

This is a **known upstream OpenWrt issue** ([openwrt/openwrt#6117](https://github.com/openwrt/openwrt/issues/6117), filed August 2019, still unresolved). The OpenWrt wiki explicitly warns: *"Flashing an OpenWrt ath79 version after 19.07.10 will brick the unit"* for AR7240-based Ubiquiti devices.

## Symptoms

- `ip link` shows `eth0` as `NO-CARRIER` / `state DOWN`
- `ethtool eth0` shows `Link detected: no`, `Auto-negotiation: off`, `Speed: 10Mb/s`
- `dmesg` shows: `connected to PHY at mdio.0:1f:04 [uid=00000000, driver=Generic PHY]`
- `swconfig dev switch0 show` reports all ports as `link:up speed:10baseT half-duplex` (bogus)
- Device works fine with AirOS or older ar71xx-based AREDN builds

## How to identify your hardware

```
cat /proc/cpuinfo | grep "system type"
```

- `Atheros AR7240 rev 2` — **affected**, needs this fix
- `Atheros AR7241 rev 1` — works fine with stock AREDN image

## The Fix

A single patch (`patches/803-ubiquiti-rocket-m-ar7240.patch`) that:

1. Replaces the DTS from `ar7241.dtsi` → `ar7240.dtsi`, putting the switch on the correct MDIO bus
2. Removes the `&eth1` compatible override that's AR7241-specific
3. Changes the SOC identifier in the image Makefile from `ar7241` → `ar7240`

The board name, compatible string, and all other identifiers remain `ubnt,rocket-m` so no other AREDN scripts need modification.

## Pre-built Firmware

The `releases/` directory contains firmware built from **AREDN 4.26.7.0** (tag `4.26.7.0`, OpenWrt `v25.12.5`) with only this patch applied:

- `aredn-n3lp--93ad9ea9-ath79-generic-ubnt_rocket-m-squashfs-factory.bin` — for TFTP/U-Boot flashing
- `aredn-n3lp--93ad9ea9-ath79-generic-ubnt_rocket-m-squashfs-sysupgrade.bin` — for sysupgrade

**⚠️ WARNING:** This image is AR7240-only. Do NOT flash on AR7241 hardware — it will break Ethernet on those devices.

## Flashing via TFTP (U-Boot recovery)

1. Set your computer to static IP `192.168.1.10/24`
2. Start a TFTP server serving the `factory.bin` file
3. Hold the reset button while powering on the Rocket M
4. Wait for the device to pull the image via TFTP
5. Release reset after LEDs indicate flash complete
6. Device will reboot to `192.168.1.1`

## Building From Source

Clone the AREDN repo at tag `4.26.7.0`, add the patch, and build:

```bash
git clone https://github.com/aredn/aredn.git
cd aredn
git checkout 4.26.7.0
cp /path/to/803-ubiquiti-rocket-m-ar7240.patch patches/
echo "803-ubiquiti-rocket-m-ar7240.patch" >> patches/series
# Edit config.mk: set PRIVATE_BUILD_VERSION to your callsign
make
```

Output: `firmware/targets/ath79/generic/*ubnt_rocket-m-squashfs-factory.bin`

## References

- [OpenWrt Issue #6117](https://github.com/openwrt/openwrt/issues/6117) — eth0 doesn't work on ath79 and AR7240 Rev 2 for ubnt-bullet-m
- [OpenWrt Wiki: Ubiquiti Airmax M](https://openwrt.org/toh/ubiquiti/airmaxm) — documents the AR7240 incompatibility
- [AREDN Forum: Rocket M5 no network](https://www.arednmesh.org/content/rocket-m5) — other users hitting the same issue

## License

Patch is GPL-2.0-or-later, consistent with OpenWrt/AREDN licensing.
