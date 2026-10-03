[Uploading README.md…]()
# Macintosh 128K Facsimile

<img src="docs/images/hero.png" alt="Macintosh 128K facsimile — Apple 50th anniversary tribute" width="100%">

![platform](https://img.shields.io/badge/platform-OpenHarmony%20%2B%20QEMU-blue)
![arch](https://img.shields.io/badge/arch-ARM%20virt-lightgrey)
![display](https://img.shields.io/badge/internal%20display-512%C3%97342-success)
![license](https://img.shields.io/badge/license-MIT-green)

**English** · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · [Español](README.es.md)

> A faithful **Apple Macintosh 128K (System 1.0 / 1984)** facsimile that boots inside
> **OpenHarmony on QEMU**. The whole GUI is a single static ARM binary that the kernel runs
> as `/init` from a tiny initrd — no OpenHarmony userspace required.
>
> Dedicated to **Apple's 50th anniversary (1976 – 2026)**.

---

## Highlights

| | |
|---|---|
| **Authentic 512×342** | The internal framebuffer is the real Mac 128K resolution, scaled ×2 to 1024×684 with black letterboxing — not stretched. |
| **Rainbow Apple menu bar** | 20 px menu bar with the 1977–1998 rainbow Apple logo, plus File / Edit / View / Special with working pull-down menus. |
| **System 1.0 window chrome** | 1 px black frame, striped title bar, close box, right + bottom scroll bars with arrows, size box, drop shadow. |
| **MacPaint** | 5 tools (pencil / eraser / line / rect / spray), 8 patterns, 260×140 canvas, drag to draw. |
| **Dock + tiling** | Bottom Dock with Disk / Notes / Paint / Trash / **Tile**; Tile arranges every open window on a grid. |
| **Clock with seconds** | `HH:MM:SS` in the menu bar, refreshed by a small-rect repaint (no full-screen flicker). |
| **Idle pointer tour** | After 5 s idle the pointer tours by itself: opens the disk window, drags it, closes it, opens About, selects the Trash, then loops. Any real input takes over instantly. |
| **Floppy insert sound** | The guest has no sound card, so it prints a cue on the serial console and a host watcher plays a synthesized floppy-insert WAV. |
| **Real mouse & keyboard** | virtio-mouse / virtio-keyboard → `/dev/input/event*`; pointer follows, double-click opens, title-bar drag, Esc closes, typing works. |

## Screenshots

| Desktop | Window | About (50th anniversary) |
|---|---|---|
| ![desktop](docs/images/01-desktop.png) | ![window](docs/images/02-window.png) | ![about](docs/images/03-about.png) |

| Menu | Tour |
|---|---|
| ![menu](docs/images/04-menu.png) | ![tour](docs/images/03-tour.png) |

## Demo video

- [`video/mac128k-demo.mp4`](video/mac128k-demo.mp4) — 1024×768, 12.5 s
- [`video/mac128k-demo.gif`](video/mac128k-demo.gif) — looping GIF

Flow: clean desktop → Dock opens the disk window → **MacPaint drawing** → Notes typing →
**multi-window tiling** → Apple menu → About.

## How it works

- **Graphics**: writes the framebuffer directly (`/dev/fb0`, 32 bpp). Double buffered
  (`scene` = content, `front` = content + pointer) with small-rect incremental blits, so
  moving the pointer never repaints the screen.
- **Input**: a static libc ARM binary (`arm-linux-gnueabi-gcc -static`) that `select()`s over
  `/dev/input/event*` and reads non-blocking.
  Lesson learned: **never printf per input event to the serial console** — at 115200 baud it
  blocks the main loop and the screen visibly stutters.
- **Boot**: the binary *is* `/init`. The kernel only needs `CONFIG_INPUT_EVDEV` and fb support.
- **Font**: a proportional Chicago-ish bitmap font auto-rasterized from DejaVuSans-Bold 11 px
  by `tools/genfont.py`.

## Build

```bash
# needs a Linaro GCC 7.5 / arm-linux-gnueabi toolchain
CC=/path/to/arm-linux-gnueabi-gcc ./build.sh
```

Produces `macgui-arm` (static ARM executable) and `initrd-macgui.img` (whose `/init` is the binary).

Regenerate the font (needs python3 + Pillow):

```bash
python3 tools/genfont.py        # writes src/font8x10.h
```

## Run

Requires **QEMU ≥ 5.1** (arm `virt` machine) and an ARM kernel `zImage-dtb`.
The reference environment uses a `linux-5.10` kernel built from an OpenHarmony 5.1 source tree.

```bash
KERNEL=/path/to/zImage-dtb ./run.sh
```

Equivalent manual command:

```bash
qemu-system-arm -M virt,highmem=off -cpu cortex-a7 -smp 1 -m 512 -name mac128k \
  -device virtio-gpu-pci,xres=1024,yres=768 \
  -device virtio-mouse-pci -device virtio-keyboard-pci -display gtk \
  -monitor unix:/tmp/mac128k-monitor.sock,server=on,wait=off \
  -qmp unix:/tmp/mac128k-qmp.sock,server=on,wait=off \
  -chardev socket,id=ser0,path=/tmp/mac128k-serial.sock,server=on,wait=off,logfile=/tmp/mac128k-serial.log -serial chardev:ser0 \
  -kernel zImage-dtb -initrd initrd-macgui.img \
  -append 'console=ttyAMA0,115200 video=1024x768 rw'
```

Optional, for the floppy sound: run `tools/macgui-sound` in another terminal while the guest runs.

## Controls

- Click the QEMU window once to give it focus, then use the mouse/keyboard.
- **Double-click** a desktop icon to open it; **drag** the title bar to move a window;
  click the **close box** or press **Esc** to close.
- Dock: Disk / Notes / Paint / Trash / **Tile**.
- Apple menu → **About This Macintosh…** for the anniversary dialog.
- Leave it idle for 5 s and the pointer starts its tour.

## Verification

Everything below was verified on the real QEMU guest (frame grabs + serial log):

| Item | Evidence |
|---|---|
| Boot | `fb 1024x768 bpp=32 | mac 512x342 x2 at +0+42` |
| Input devices | `event0 'QEMU Virtio Mouse'`, `event1 'QEMU Virtio Keyboard'` |
| Clock seconds | two frames 1.6 s apart differ by 60 px in the clock region |
| Pointer accuracy | after homing, 50 paced steps moved the pointer to exactly `x=500 y=300` |
| Double-click → window | `click x=956 y=248` → `win kind=1` → `OPEN icon=0` |
| MacPaint | stroke landed on the canvas (652 black pixels at the expected position) |
| Dock | dock strip renders (black 2564 / white 12636) |
| Sound cue | `MACGUI7: SOUND disk-insert` fired and the host watcher played the WAV |
| Idle tour | `tour start` → open → drag → close → `about` → `dlg ok` → loop |

## License

MIT — see [LICENSE](LICENSE).
Macintosh and the Apple logo are trademarks of Apple Inc. This is a **non-commercial tribute**.

