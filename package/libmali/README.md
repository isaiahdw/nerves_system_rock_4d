# libmali (Mali-G52 userspace)

Proprietary ARM Mali userspace (EGL/GLESv2/GBM) for the RK3576 Mali-G52,
DDK g24p0, plain-GBM variant (no X11/Wayland deps). Pairs with the
in-kernel Mali Bifrost module (`CONFIG_MALI_BIFROST`) that creates
`/dev/mali0`.

Forked from Buildroot's `rockchip-mali` package, which is hardcoded to G31.

## The blob (downloaded — 54 MB)

`libmali-bifrost-g52-g24p0-gbm.so` is fetched at build time from Rockchip's
public libmali distribution — the `libmali` branch of
[JeffyCN/mirrors](https://github.com/JeffyCN/mirrors/tree/libmali), pinned
by commit in `libmali.mk` and verified against `libmali.hash` (the public
blob is byte-identical to the one in the vendor SDK's `external/libmali`).

Headers (`include/`) and pkgconfig (`pkgconfig/`) are committed.

## DDK version note

The kernel's Mali kmod reports DDK g25p0; this userspace is g24p0
(the newest G52 GBM variant available). Adjacent DDK releases share the
kbase UABI, so they are expected to be compatible — verify with `kmscube`.
If the GPU fails with a kbase version mismatch, try another DDK variant
from the mirror's `lib/aarch64-linux-gnu/` or switch to the matched
out-of-tree `mali-driver` kmod.

## Display notes for EGL/GBM UI stacks

Two known gotchas on the RK3576 VOP apply to any EGL/GBM UI stack driving
the HDMI output:

1. Linear buffers, not AFBC: libmali allocates ARM AFBC-compressed GPU
   buffers by default, but vop2 primary planes may not scan out AFBC
   (boot log: `unsupported AFBC format`). The result is a blank display
   even though rendering succeeds. `kmscube -m 0` forces the linear
   modifier. A UI stack must likewise use linear scanout buffers (GBM
   `LINEAR` modifier, or the libmali AFBC-disable env var).

2. Release the framebuffer console: fbcon owns the display (this
   system keeps the kernel console on HDMI). Unbind it before a KMS app
   takes over:
   `for f in /sys/class/vtconsole/vtcon*/bind; do echo 0 > $f; done` —
   or have the UI app do so on launch.

## Licensing

The blob is proprietary ARM/Rockchip software (`LIBMALI_LICENSE =
PROPRIETARY`). No license text ships with Rockchip's public libmali
distribution — the download is the bare `.so` — so `LIBMALI_LICENSE_FILES`
is deliberately unset and Buildroot's legal-info warns accordingly. The
blob is not committed to this repository; it is fetched at build time
from Rockchip's public mirror and redistributed here only inside built
firmware images, the same arrangement Rockchip BSPs and Armbian use.

## Open alternative (Panfrost)

Mesa's open Panfrost driver supports the Mali-G52 and would remove the
blob, but Mesa 25's panfrost requires building LLVM, which needs a Docker
build VM with >12 GB RAM. Revisit when that's available.
