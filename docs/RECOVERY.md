# Recovery

If a flash goes wrong, force firmware-update/recovery mode and reflash the
original, unmodified update file (the installer keeps a copy — see the root
[README](../README.md)):

- **Pedalboard**: hold footswitches **1 and 8** (leftmost, counting left to
  right) while powering on. See
  [this video](https://www.youtube.com/watch?v=6H90kbOCJG8) for a walkthrough.
- **MX5**: hold the **first two** footswitches while powering on.
- **Gigboard**: no footswitch-hold combo re-flashes firmware, but a
  **factory reset** is available: power off, hold footswitches **3 and 4**
  together, then power on while still holding them. Keep holding until the
  screen shows "reverting" — this wipes all user memory, custom rigs, and
  IRs and restores factory defaults, but does **not** re-flash firmware, so
  it won't undo a bad NAM-mod flash by itself. Back up anything on the USB
  drive first. The stock firmware-update path remains UI-driven only
  (Global Settings → Firmware Update on the touchscreen).

Tested on real HeadRush Pedalboard, MX5, and Gigboard devices: NAM inference
works. Pedalboard and MX5 can be safely recovered back to stock firmware via
the footswitch combos above; Gigboard has no footswitch combo that re-flashes
firmware — only the UI-driven update path and the factory-reset combo above
(which clears user data but not firmware). Proceed at your own risk.
