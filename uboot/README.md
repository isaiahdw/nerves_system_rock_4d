# U-Boot boot chain

The ROCK 4D boot ROM does not boot from the SD card. Its boot order is
SPI NOR → UFS → USB (maskrom), and every standard ROCK 4D ships with a
16 MB SPI NOR on board. U-Boot therefore lives in the SPI NOR and the SD
card only carries the Nerves environment, kernels and root filesystems:

| Blob                      | Location                  | Contents                                                         |
| ------------------------- | ------------------------- | ---------------------------------------------------------------- |
| `u-boot-rockchip-spi.bin` | SPI NOR, offset 0         | idblock (DDR init + SPL) + FIT (U-Boot proper + BL31) at 0x60000 |
| `uboot-env.bin`           | SD card, 15 MB (0xF00000) | Nerves environment (built from `uboot.env`)                      |

The committed `u-boot-rockchip-spi.bin` is mainline U-Boot v2026.01
(`rock-4d-rk3576_defconfig`, added upstream in October 2025) plus the
Rockchip rkbin blobs (RK3576 DDR init v1.12, BL31 v1.24 — there is no
open-source DRAM init or BL31 for this SoC), with the Nerves environment
support added:
`CONFIG_ENV_IS_IN_MMC=y`, `CONFIG_ENV_OFFSET=0xF00000`,
`CONFIG_ENV_SIZE=0x20000`, `CONFIG_SYS_MMC_ENV_DEV=0` (the SD card).

Rebuild it reproducibly with `scripts/build-uboot.sh` (Docker; pinned
U-Boot tag + rkbin commit). The script also produces `u-boot-rockchip.bin`
(the same boot chain laid out for block storage, useful for future UFS
experiments); that variant is not committed.

Licensing of the committed binaries: U-Boot itself is GPL-2.0-or-later
(corresponding source: the pinned tag at https://source.denx.de/u-boot/u-boot
plus this script). The DDR-init and BL31 components inside both
`u-boot-rockchip-spi.bin` and `rk3576_spl_loader_v1.09.108.bin` are
Rockchip proprietary blobs from
[rockchip-linux/rkbin](https://github.com/rockchip-linux/rkbin),
redistributable per `LICENSES/LICENSE.rockchip-rkbin`.

## Boot and automatic revert

U-Boot runs the `nerves_init`/`nerves_boot` scripts from the environment
block on the SD card and boots the active slot's `Image.<slot>` + dtb
directly (`root=PARTUUID=<slot GUID>`). Automatic revert follows the
standard Nerves model:

- U-Boot: with `nerves_fw_autovalidate=0`, new firmware boots once
  (`booted` 0→1) leaving `nerves_fw_validated=0`. If the next boot still
  sees `validated=0`, U-Boot boots the other slot.
- Application: calls `Nerves.Runtime.validate_firmware()` once healthy
  (sets `validated=1`), or the update reverts.

If `nerves_boot` ever fails, `bootcmd` falls through to `bootflow scan -lb`,
which boots `extlinux/extlinux.conf` from the FAT partition. A missing or
corrupt environment block makes U-Boot fall back to its compiled-in default
environment, which does the same — a bad environment cannot brick the
device.

## uboot.env — the shared firmware/boot environment

`uboot.env` is compiled into `uboot-env.bin` and written raw at 15 MB
(`UBOOT_ENV_OFFSET`, see `fwup_include/fwup-common.conf` and
`rootfs_overlay/etc/fw_env.config`). It is a single fw_env block shared by
three parties: U-Boot reads it to pick the boot slot, `nerves_runtime`/
`fwup` read and write the `nerves_fw_*` firmware metadata, and `boardid`
reads the serial number.

## Flashing the SPI NOR (one-time provisioning, via maskrom)

A new board ships with Radxa's U-Boot in the SPI NOR. Radxa's U-Boot does
extlinux/distro boot, and this system's SD card keeps a valid
`extlinux/extlinux.conf` at all times — so a fresh board boots the Nerves
SD on the factory bootloader, minus the Nerves env integration: no
automatic revert of unvalidated firmware (upgrades still switch slots via
the extlinux.conf swap). That is fine for bench work; flash ours when you
want the full boot chain.

Flashing is done over USB from the host — no OS on the board involved.
Hold the MASKROM button while applying power. The maskrom USB device
appears on the upper USB 3.0 Type-A OTG port (top-left, nearest the
corner) — use a USB-A↔A or USB-A↔C data cable to the host. Then, from
this directory:

```sh
rkdeveloptool db rk3576_spl_loader_v1.09.108.bin  # bootstrap the loader
rkdeveloptool cs 9                                # switch storage to SPI NOR
rkdeveloptool wl 0 u-boot-rockchip-spi.bin        # write at offset 0
rkdeveloptool rd                                  # reboot
```

Verify on the serial console: the U-Boot banner should read v2026.01.

`rk3576_spl_loader_v1.09.108.bin` is the RK3576 maskrom download loader,
built from the Rockchip vendor SDK's U-Boot (`./make.sh rk3576` packs it
from the rkbin SPL/DDR components). If `cs` is unsupported by your
rkdeveloptool build, use a current build from
https://github.com/rockchip-linux/rkdeveloptool or Rockchip's RKDevTool
on Windows.

Maskrom is also the recovery path if a NOR write is ever interrupted —
the BootROM always runs first, so the board is not brickable this way.

Serial console for watching the boot: UART0 on the 40-pin header — pin 8
(TX), pin 10 (RX), pin 6 (GND), 1500000 8N1, no flow control.
