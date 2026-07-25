# Changelog

## v0.1.0

Initial release. Verified on hardware 2026-07-26: full boot on the factory
(Radxa) SPI bootloader via the extlinux path, all subsystem checks passing
(ethernet, onboard WiFi/BT, GPU, HDMI, watchdog, RTC, audio, app
partition, env).

Hardware support at release:

- Boot: mainline U-Boot v2026.01 (`rock-4d-rk3576_defconfig` + rkbin DDR
  init/BL31) in the onboard SPI NOR — the ROCK 4D boot ROM cannot boot from
  SD, so the SPI flash step is one-time provisioning (see `uboot/README.md`).
  Nerves U-Boot-environment A/B slots on the SD card with automatic revert,
  delta firmware updates (fwup >= 1.12.0 on device), extlinux fallback path.
- Kernel: armbian/linux-rockchip `rk-6.1-rkr5.1` pinned by commit SHA,
  in-tree `rk3576-rock-4d-spi` device tree, `rockchip_linux_defconfig` +
  Nerves fragment. One board dts patch
  (`linux/0001`): enable the watchdog for nerves_heart, and alias `mmc0`
  to the SD card so it enumerates as `/dev/mmcblk0` (the vendor dtsi
  aliases mmc0 to the disabled eMMC).
- Ethernet: GbE on gmac0 with RTL8211F PHY (`CONFIG_REALTEK_PHY` added on
  top of the vendor defconfig, which lacks it).
- Display: HDMI on VOP with the framebuffer console enabled; Mali G52 MC3
  via vendor kmod + libmali (EGL/GLES/GBM), kmscube smoke test. No UI
  toolkit ships in the system — the application brings its own DRM/KMS
  stack; eudev + libinput for USB input.
- Audio: ES8388 codec (headphone jack) and HDMI audio via ALSA.
- SPI NOR exposed as the `loader` mtd + flashcp for on-device bootloader
  updates.
- Watchdog-backed heart, HYM8563 RTC, USB3/USB2 hosts, USB gadget ethernet,
  f2fs application partition.

- WiFi/BT: onboard Quectel FCU760K (AIC8800D80, USB) via the out-of-tree
  radxa-pkg/aic8800 driver + firmware (`package/aic8800`, Radxa's full
  patch series applied); wlan0 + BlueZ hci0, modalias-autoloaded.

Not yet supported: PCIe/NVMe (untested), UFS modules, NPU, MIPI DSI/CSI.
