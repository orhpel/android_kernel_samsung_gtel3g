# android_kernel_samsung_gtel3g

Kernel source for Samsung Galaxy Tab 4 10.1 (SM-T560/T561: `gtel3g` / `gtelwifi`)
running LineageOS 17.1 (Android 10). Used by the hybrid11–hybrid27 boot images
and the `lineage-17.1-gtel3g/gtelwifi` ROMs.

- Base: `lineage-16.0` branch @ `327dafe8` (3.10.108)
- Defconfig: `arch/arm/configs/gtel3g_defconfig`
- Toolchain (mandatory): `arm-eabi-4.8` from AOSP
  (`remote aosp-gs`, branch `nougat-mr1-release`, commit `26e93f6a`).
  Builds with gcc 4.9 do **not** boot on this device (12.2 MB+ Image,
  logo hang). A correct build yields `Image` = **12056864 bytes**.

## Build

```bash
export PATH=<android-root>/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8/bin:$PATH
make ARCH=arm CROSS_COMPILE=arm-eabi- O=<out> gtel3g_defconfig
make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-eabi- O=<out> zImage dtbs
```

`bc`, `flex`, `bison` are required on the host (this tree uses the
backported `kernel/timeconst.bc`, not the `.pl` script).

## Local changes on top of 327dafe8

1. `arch/arm/mm/alignment.c` — always emulate unaligned accesses
   (`UM_FIXUP`, strip `UM_SIGNAL`): SPRD ION heaps are mapped
   strongly-ordered, killing SoundPool (`str` Thumb-1) and crashing
   `media.swcodec` on Vorbis endings.
2. `arch/arm/mm/alignment.c` — emulate Thumb-2 word LDR/STR by rewriting
   to the ARM equivalent (upstream refuses them; same trick as the T3
   PUSH/POP case above it). Fixes the remaining `media.swcodec` SIGBUS.
3. Display earlysuspend pared down to the HWC/FBIOBLANK path only
   (`sprdfb_main.c`, `rt8555-backlight.c`, `gsp_drv.c`,
   `sprd_iommu_gsp.c` registration commented out): with Android 10's
   `android.system.suspend`, the legacy chain blanked the panel behind
   SurfaceFlinger and wedged touch/Wi-Fi/DDR scaling.
4. `zt7554_ts.c` — touch suspend/resume via fb notifier instead of
   earlysuspend (the path Android 10 actually exercises).
5. `net/netfilter/xt_IDLETIMER.c` — emit `INTERFACE=`/`TIME_NS=` instead
   of `LABEL=`: Android 10 netd `ParseInt(NULL)` → SIGSEGV → zygote restart.
6. `drivers/media/sprd_gsp/gsp_config_if.c`, `gsp_drv.c` — bound the
   formerly infinite `gsp_busy` poll (`GSP_Wait_Finish`, ~0.5 s budget,
   `-ETIMEDOUT`); callers abort the frame loudly instead of hanging the
   caller forever in kernel (unkillable, undumpable, permanent black
   screen). See `docs/HYBRID27_GSP_TIMEOUT.md` in the bringup repo.
7. `drivers/video/sprdfb/sprdfb_dispc.c` — escape hatch in
   `dispc_stop_for_feature()` busy poll.

## Version-string note

Shipped binaries report `3.10.108-g327dafe8-dirty` (gcc 4.8). The `-dirty`
is from the development tree; this commit contains byte-identical sources
(the only post-commit tree edit was a CRLF→LF normalization of
`gsp_config_if.h`, semantically null).
