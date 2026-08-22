# MX5 remote screen

Mirror the HeadRush MX5's display on a PC over USB, and operate it with the
mouse. **Confirmed working on real MX5 hardware** (firmware 2.7).

Requires the second USB serial channel from [`../mx5_usb_console`](../mx5_usb_console)
(`ttyGS1`), so flash that first.

```
                MX5                                    PC
   /dev/fb0 ──► mx5_screen_server ──► ttyGS1 ═══USB═══► mx5_viewer.py ──► window
   /dev/uinput ◄──────────────────────  ttyGS1 ◄═══════════════════════ mouse
```

## Running it

### Installed (recommended)

`./build_install_screen.py` bakes the server into the firmware and enables it at
boot, so the screen share is live as soon as the device is up — no login, and no
re-pasting the binary after every reboot. Chain it after the gadget mod:

```sh
../mx5_usb_console/build_usb_console.py stock.img a.img
./build.sh                                            # produce the ARM binary
./build_install_screen.py                a.img     final.img
```

Then open a viewer. Two are provided; they speak the same protocol and are
interchangeable.

**Browser (no dependencies)** — open `mx5_viewer.html` in Chrome or Edge, click
**Connect**, and pick the *second* MX5 serial port (the data channel; the first
is the shell). Uses the Web Serial API, so there is nothing to install.
Not available in Firefox or Safari, which don't implement Web Serial. It needs a
secure context: opening the file directly works in Chrome, but if
`navigator.serial` is missing, serve the directory over localhost instead.

**Python**

```sh
pip install pyserial pygame
python mx5_viewer.py COM7            # or /dev/ttyACM1 on Linux/macOS
```

Left-click to tap; click-drag to swipe. **Arrow keys turn the encoder knob**
(up/left = counter-clockwise, down/right = clockwise) and **space/enter presses
it**. Both viewers support this.

**Idle cost is negligible**: the server holds no flow-control credit until a
viewer asks for a frame, and it never reads the framebuffer without credit — it
just wakes briefly to check for a client. It is also `Nice=10` and pinned to
CPU3, away from the audio path.

The service is ordered `After=usb-console.service` so systemd stops it *before*
the USB gadget is torn down. That ordering matters: a process still holding
`/dev/ttyGS1` open while the gadget is removed can hang shutdown — which is
exactly what happens if you background the server from a shell instead, where
systemd knows nothing about it.

### By hand (for iterating)

Paste the deploy blob into the root shell (see below), then:

```sh
taskset -c 3 /tmp/mx5_screen_server -r 15 <> /dev/ttyGS1 >&0 2>/tmp/srv.log &
```

`<> /dev/ttyGS1 >&0` opens the port read-write as both stdin and stdout, so
frames go out and touch events come back on the same channel. Kill it with
`pkill -f mx5_screen_server` before shutting down, for the reason above.

## Building and deploying

`./build.sh` cross-compiles with the repo's Bootlin glibc-2.31 toolchain (the
device has glibc 2.32) and writes `mx5_screen_server.deploy.sh` — a paste-able
`gzip`+`base64` blob, ~7 KB. Paste that into the device's root shell and busybox
reconstructs the binary in `/tmp`. No reflashing is needed to iterate, which is
what makes this practical to develop.

`/tmp` is tmpfs, so re-paste after a reboot.

## How it works

**Capture — `/dev/fb0`.** Qt uses the `eglfs_mali` backend, which presents
through fbdev, so the framebuffer really does hold the live composited screen.
(Had it been `eglfs_kms` this approach wouldn't work at all.) The framebuffer is
480x800 32bpp and **triple buffered** (virtual height 2400), so the server reads
`FBIOGET_VSCREENINFO.yoffset` every frame to find the currently visible buffer —
reading a fixed offset yields stale or torn frames.

**Pixels are BGR*X*.** The fourth byte is padding and reads as `0x00` on this
device. Treating it as alpha makes every pixel fully transparent — which renders
a completely black window while the data is in fact perfect.

**Rotation.** The panel is mounted rotated: the framebuffer is portrait, the UI
is landscape. The viewer rotates for display and applies the exact inverse to
mouse coordinates.

**Encoding.** The screen is split into 32x32 tiles; only tiles that changed since
the last frame are sent, so an idle screen costs nothing. Tiles are RLE-encoded
over 32-bit pixels, which suits this UI — a flat tile goes from 4096 bytes to 6.
Tiles that don't compress fall back to raw, so a tile is never larger than raw.

**Flow control.** The server only sends a frame while it holds credit, granted by
the viewer's ack after each frame. This is essential, not an optimization: delta
frames cannot be dropped to catch up (each depends on the last), so without
pacing the device outruns the host and latency grows without bound. While the
server holds no credit it leaves its shadow buffer untouched, so changes
accumulate and coalesce into the next frame it may send.

**Resync.** Every frame carries a self-describing header with a magic, so a
viewer can attach, detach and reattach at any time — a serial port has no
connection semantics, so there's no other way to know a client appeared. On any
malformed data the viewer drops its buffer, asks for a keyframe and resyncs
rather than dying.

**Input.** Touches are injected via `/dev/uinput` as an absolute multitouch
device, so they look like the real ili2116 touchscreen to Qt.

**Encoder.** The knob is *not* an input device — it's the control-surface MCU
sending MIDI over a serial link (`snd-serdev-midi`), which Evil consumes through
the ALSA sequencer. So uinput can't reach it; the server has to become a MIDI
source. Evil creates one ALSA seq client per MIDI device:

```
client  20: 'HG04 Control Surface' [kernel]  ──▶ client 129: 'Midi::In::HG04 Control Surface MIDI 1'
```

That client-129 port accepts writes, and Evil attributes events by *which of its
own ports* they arrive on — so events we send there are indistinguishable from
the real control surface, and the stock assignment file maps them for free:

| Message | Assignment target |
|---|---|
| CC 3 | `JogOutput` → `/Engine/PushEncoderCtrl/Encoder` (turn) |
| Note 4 | `PressAndHoldOutput` → `/Engine/PushEncoderCtrl/EncoderEnter` (push) |

The port is located by **name**, since seq client numbers are assigned
dynamically. `libasound` is `dlopen`ed, so the screen share still works if it's
unavailable — only the encoder is lost.

`JogOutput`'s relative-CC convention isn't documented in the firmware, and the
common encodings disagree on how negative is represented, so `-j` selects it:
`0` two's complement (default, −1 → 127), `1` signed bit (−1 → 65), `2` binary
offset (0 → 64). If the knob turns the wrong way or does nothing, try the others.

**Scheduling.** Pin to a core away from audio. On this device CPU0 services the
I2S DMA IRQ plus ~20 Evil threads and CPU2 carries ~14 (the DSP pool), while CPU3
has one — hence `taskset -c 3`. (The RK3288 is quad-core; a claim elsewhere in
this repo that the device has a single core is incorrect.)

## Performance notes, learned the hard way

The link sustains **~12 MB/s** (USB high-speed CDC-ACM), which is ~8 full
uncompressed frames per second before any compression — so no image codec is
needed on the device.

Two host-side mistakes each destroyed performance, and both were measured:

1. **Driver buffer too small.** Leaving pyserial's Windows RX buffer at its 4 KB
   default throttled 12 MB/s down to 0.6 MB/s. Hence `set_buffer_size`. The
   browser viewer has the same hazard in a different guise: Web Serial's
   `open({bufferSize})` defaults to **255 bytes**, so it passes 1 MB explicitly.
2. **Asking `read()` for more than needed.** `read(n)` blocks until n bytes
   arrive *or the timeout expires*, so requesting 64 KB "for batching" makes
   every read wait out the timeout instead of returning available data. This
   produced multi-second lag, and mid-frame timeouts then caused stream
   desyncs that looked like device-side corruption. `Reader._fill` now asks for
   exactly the deficit and then drains `in_waiting`, which batches without ever
   waiting. Web Serial avoids this trap by construction -- its `read()` returns
   whatever has arrived rather than waiting for a requested count.

## Limitations

* Animated regions (tuner needle, level meters) change every frame and will use
  real bandwidth; this is built for operating the UI, not watching animation.
* Touch injection relies on Qt picking up a uinput device created *after* Evil
  started. It works on this firmware, but a device created later isn't
  guaranteed to be enumerated by an already-running Qt app.
* Touch injection and the framebuffer are both read/written as root; the server
  runs as root out of necessity.
