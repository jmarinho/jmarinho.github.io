---
title: Resurrecting a 2010 Toshiba Folio 100 Tablet as a Linux Calendar Display
date: 2026-04-05
layout: default
---


# Resurrecting a 2010 Toshiba Folio 100 Tablet as a Linux Calendar Display

> *How I turned a forgotten Android tablet into a functional Linux-based project tracker using Debian, a custom kernel initramfs, and a locally-served web app — with working touchscreen.*

---

## Background

The **Toshiba Folio 100** (PA3895E-1ET1) is a 10-inch Android tablet from 2010, powered by NVIDIA's Tegra 2 (dual-core ARM Cortex-A9 at ~1GHz) with 360MB RAM and a 1024×600 display. It was a commercial flop, quickly abandoned by Toshiba, and hasn't received any software updates since 2012. Most people threw theirs away.

I had one sitting in a drawer. The goal: turn it into a dedicated wall-mounted project calendar display running Linux — no Android, no cloud, just a locally served HTML/JS web app visible on the screen.

This is the story of how that actually worked.

---

## The Hardware

| Component | Details |
|-----------|---------|
| SoC | NVIDIA Tegra 2 (T20), Tegra SKU 8 Rev A02 |
| CPU | Dual-core ARM Cortex-A9 |
| RAM | 360MB (physical framebuffer at 0x16800000) |
| Display | 1024×600, 16-bit framebuffer |
| Storage | 16GB eMMC |
| Touchscreen | eGalax (I2C), multitouch |
| WiFi | Atheros AR6003 (ATH6K_LEGACY) |
| USB | EHCI Tegra, supports ADB gadget |

The device boots via NVIDIA's proprietary `nvflash` tool over USB in APX mode (hold power, tap Vol- during Toshiba logo). The partition table lives on the eMMC and the kernel boots from the `LNX` partition at sector 659968 with 2048 bytes/sector.

---

## The Plan

Instead of trying to port a full Android ROM (done, abandoned in 2013 by the community) or fight with a mainline kernel that wouldn't boot (we tried — DTB memory size mismatch: paz00.dtb assumes 512MB, device has 360MB), we took a different approach:

1. Keep the original **CyanogenMod 10 recovery kernel** (3.1.10+, 2012) — it's proven to boot
2. Replace the **initramfs** with a custom one that mounts Debian from an SD card
3. Run **Debian 12 Bookworm armhf** in a chroot from the SD card
4. Launch **Xorg with fbdev** driver pointing at `/dev/graphics/fb0`
5. Serve the calendar app with **Python's HTTP server** and display it in **surf**

---

## Stage 1: Getting nvflash to Work

Flashing requires getting into the APX mode of the tablet.
US versions of the tablet allow a volume up/down combo that gets into that mode.
For the EMEA version, we need to access the button behind the cover, as you can see in [Figure 1](#fig-folio_img1).

<a id="fig-folio_img1"></a>

![Folio 100 debug buttons behind cover.]({{ "_entries/imgs/toshiba_img1.jpg" | relative_url }}){#fig:folio_img1}

The Folio 100 uses NVIDIA's proprietary flashing tool. Getting it working on a modern Linux box required:

- Running the x86 `nvflash` binary via `qemu-x86_64` on an ARM64 host
- Using the correct `odmdata` value: `0x800c0075` (critical — using `0x000c0075` breaks USB)
- Entering APX mode correctly: power on → Toshiba logo → press Power 4× then Vol-
- Reading the real partition table first to get correct sector offsets

The `LNX` partition (boot) is at sector 659968, size 4096 sectors. The `SOS` partition (recovery) is at sector 1188864, size 2560 sectors.

```bash
# Flash boot partition
./nvflash --bct new.bct --setbct --bl bootloader.bin \
  --odmdata 0x800c0075 \
  --rawdevicewrite 659968 4096 boot_debian.img --go
```

Before touching anything, we backed up both partitions:

```bash
./adb shell dd if=/dev/block/mmcblk0p4 of=/cache/lnx.img
./adb shell dd if=/dev/block/mmcblk0p6 of=/cache/sos.img
```

---

## Stage 2: Understanding the Recovery Kernel

The recovery kernel is a monolithic ARM zImage compressed with LZMA, with no loadable modules and no device tree (pre-DT era, uses ATAGs). Key findings from `/proc/config.gz`:

- `CONFIG_MODULES=y` — modules supported, but none shipped with recovery
- `CONFIG_FB_SYS_FOPS` is **not set** — this matters a lot for X11 (more on this later)
- `CONFIG_FB_TEGRA=y`, `CONFIG_TEGRA_NVMAP=y` — custom Tegra framebuffer
- `CONFIG_ATH6K_LEGACY=m` — WiFi as a module, but no .ko file anywhere
- `CONFIG_TOUCHSCREEN_EGALAX=y` — touchscreen built-in
- `CONFIG_USB_G_ANDROID=y` — ADB gadget support built-in

The framebuffer physical address is `0x16800000`, which is also the upper limit of usable RAM (360MB = 0x16800000).

---

## Stage 3: Building the Custom Boot Image

Android boot images have a specific format: a 2048-byte header followed by the kernel, then the ramdisk, all page-aligned.

We extracted the recovery kernel from `p6.img` (the SOS partition backup) and built a new initramfs around it.

The initramfs (built from the original recovery ramdisk) handles:

```sh
# Mount SD card
mount -t ext4 /dev/block/mmcblk1p1 /mnt/debian

# Bind mounts into chroot
mount --bind /dev /mnt/debian/dev
mount --bind /proc /mnt/debian/proc
mount --bind /sys /mnt/debian/sys
mount -t devpts devpts /mnt/debian/dev/pts
mkdir -p /mnt/debian/dev/shm
mount -t tmpfs tmpfs /mnt/debian/dev/shm

# Start Xorg inside chroot
chroot /mnt/debian /bin/sh -c '... Xorg :0 -novtswitch -sharevts -nolisten tcp &'
```

The SD card had to be formatted with old-style ext4 features — the 2012 kernel doesn't support `extent`, `metadata_csum`, `64bit`, or `orphan_file`:

```bash
mkfs.ext4 -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,sparse_super,large_file /dev/sdb1
```

---

## Stage 4: Debian on SD Card

We used `debootstrap` with `--variant=minbase` for a minimal footprint (~965MB used of 3.7GB):

```bash
debootstrap --arch=armhf --foreign --variant=minbase bookworm /mnt/debian \
  http://deb.debian.org/debian
```

The second stage ran on the tablet itself via ADB — native ARM execution, no qemu needed:

```bash
adb shell chroot /mnt/debian /debootstrap/debootstrap --second-stage
```

Then installed X essentials:

```bash
apt-get install -y --no-install-recommends \
  xserver-xorg-core \
  xserver-xorg-video-fbdev \
  xserver-xorg-input-evdev \
  openbox \
  surf \
  xinput \
  python3
```

---

## Stage 5: The Framebuffer Problem

This was the trickiest part. Xorg with the `fbdev` driver loaded fine and correctly identified the display:

```
(II) FBDEV(0): using /dev/graphics/fb0
(II) FBDEV(0): Virtual size is 1024x600 (pitch 1024)
(II) FBDEV(0): hardware: tegra_fb (video memory: 8192kB)
```

But the screen stayed black. The reason: **`CONFIG_FB_SYS_FOPS` is not set** in the recovery kernel. This option enables standard file operations (including mmap) on the framebuffer for userspace. Without it, X's shadow framebuffer renders internally but never flushes to the hardware.

We verified this by comparing `/proc/PID/maps` for the recovery binary (which draws correctly) vs Xorg — both mmap `fb0` at physical address `0x16800000` identically. The difference was elsewhere.

The actual fix came from understanding the **Tegra display controller's double-buffering**: the hardware uses a pan/flip mechanism. Writing to `/sys/class/graphics/fb0/pan` triggers the display controller to refresh. Without this, rendered frames sit in the framebuffer but never appear on screen.

```bash
# Pan trigger daemon — runs every second to flush display
while true; do
    echo 0,0 > /sys/class/graphics/fb0/pan
    sleep 1
done &
```

Once we added this loop, X rendered correctly.

---

## Stage 6: Touchscreen Calibration

The eGalax touchscreen (event2) sends `ABS_MT_POSITION_X/Y` multitouch events with a range of 0–32767 on both axes. Xorg's evdev driver defaults to assuming the range matches the screen resolution (0–1024, 0–600), which maps all touches to a tiny corner of the screen.

The fix — set the evdev axis calibration at runtime after X starts:

```bash
xinput set-prop Touchscreen "Evdev Axis Calibration" 0 32768 0 32768
```

This is called from `/usr/local/bin/fix-touch.sh` which runs from the initramfs script after X initialises. The `Calibration` option in xorg.conf is ignored for multitouch axes — runtime `xinput` is the only thing that works.

---

## Stage 7: The Calendar App

The calendar is a self-contained Python HTTP server + single-page HTML/JS app:

**`server.py`** — serves `index.html` and handles GET `/config` and POST `/save` to read/write a JSON config file.

**`config.json`** — stores start date, end date, and milestones:

```json
{
    "start_date": "2026-03-01",
    "end_date": "2026-08-20",
    "milestones": [
        {"date": "2026-04-15", "label": "Beta Release"}
    ]
}
```

**`index.html`** — renders a full calendar from start to end date:
- Past days get a red ✕ overlay
- Milestone days are highlighted in yellow
- Crossed milestones show yellow background with red ✕ overlay
- Tap any in-range day to toggle milestone / set label
- Config panel for editing dates and milestones

The browser is `surf` (WebKit-based, minimal), launched fullscreen pointing at `http://localhost:8080`.

---

## The Full Boot Sequence

```
nvflash → LNX partition → recovery kernel (3.1.10+)
  → initramfs runs start_debian.sh
    → mount SD card ext4
    → bind mount /dev /proc /sys devpts tmpfs
    → stop recovery service
    → chroot: start Xorg with fbdev on /dev/graphics/fb0
    → chroot: start openbox window manager
    → sleep 5s
    → fix-touch.sh: xinput calibration 0 32768 0 32768
    → chroot: python3 /opt/calendar/server.py
    → sleep 2s
    → chroot: surf http://localhost:8080
    → pan trigger loop (1s interval)
```

Boot to usable calendar display: approximately 15 seconds.

---

## What Didn't Work (and Why)

**Mainline kernel**: paz00.dtb specifies 512MB RAM but the device has 360MB. Kernel crashes before any output. Even after patching the memory node, the machine ID passed by the bootloader doesn't match what mainline expects.

**WiFi**: The ATH6K_LEGACY module (.ko) isn't included in the recovery partition and we'd need to compile it against the exact kernel build. The kernel was built by `artem@amakhutov` on 14 Oct 2012 — getting an exact-match toolchain is non-trivial.

**switch_root**: The recovery busybox doesn't include `switch_root`, and the Debian version is an armhf binary that the recovery shell can't execute directly. The chroot approach works just as well.

**mmap framebuffer from Python**: `mmap()` on `/dev/graphics/fb0` from Python silently writes to a shadow buffer that never reaches the display, due to missing `CONFIG_FB_SYS_FOPS`. Direct `write()` via `dd` works fine.

---

## Bill of Materials

- Toshiba Folio 100 tablet (PA3895E-1ET1)
- 4GB microSD card
- USB keyboard (for initial setup)
- Linux box with ADB and nvflash
- A lot of patience

---

## Key Takeaways

1. **Old kernels are findable**: The exact config from a 2012 kernel survives in `/proc/config.gz`
2. **Recovery partitions are goldmines**: A working kernel + ramdisk you can extend
3. **Chroot beats switch_root** when your userspace tools are incompatible
4. **Tegra framebuffer needs pan**: The display controller won't update without it
5. **evdev Calibration doesn't apply to multitouch axes** — use `xinput set-prop` at runtime
6. **debootstrap minbase** is surprisingly capable at ~300MB

---

## Source

All scripts, config files, and the calendar app are available in this repository.

The calendar displays a project timeline from March 2026 to August 2026, with:
- Automatic red ✕ on past days
- Configurable milestones shown in yellow
- Tap-to-configure UI
- Persistent JSON config on the SD card

*Total project time: several evenings of increasingly obsessive debugging.*

---

*If you have a Folio 100 in a drawer somewhere — it still works.*
