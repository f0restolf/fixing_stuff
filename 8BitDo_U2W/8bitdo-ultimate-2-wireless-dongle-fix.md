# 8BitDo Ultimate 2 Wireless — 2.4GHz Dongle Game Detection Fix

**System:** Nobara Linux 43, kernel 6.17.x, Plasma 6 / Wayland
**Hardware:** 8BitDo Ultimate 2 Wireless Controller + included 2.4GHz dongle

---

## Issue

The controller works correctly over Bluetooth. Via the 2.4GHz dongle in its default mode, the controller is detected by Steam (Big Picture sees it, controller navigates the UI), but is **not** picked up by games — including:

- SDL2-based, Linux-native games with built-in controller support
- Unity-based games running through Proton with built-in controller support

Disabling Steam Input per-game does not resolve it. Switching the game between Proton and native Linux binaries (via Steam Linux Runtime 1.0 / Scout) does not resolve it.

---

## What was confirmed

| Connection path | Big Picture | Game input |
|---|---|---|
| Bluetooth | ✓ | ✓ |
| 2.4GHz dongle — default mode | ✓ | ✗ |
| 2.4GHz dongle — D-input mode | ✓ | ✓ |

Raw input flow was verified working in both dongle modes:

- `evtest` shows all button and axis events firing
- `jstest /dev/input/js0` shows the same

So the kernel side, HID stack, and udev permissions were never the problem — input was reaching userspace correctly in all cases. The break was in how userspace gamepad APIs classify the device.

---

## Root cause

The 2.4GHz dongle has two firmware modes that expose the controller differently to the kernel.

### Default mode — USB ID `2dc8:310b`

Exposes a **composite HID device** with three sibling input nodes:

```
N: Name="8BitDo Ultimate 2 Wireless Controller"          → event10 / js0   (gamepad)
N: Name="...Controller for PC Keyboard"                  → event14         (keyboard + sysrq + leds)
N: Name="...Controller for PC Mouse"                     → event15 / mouse2 (mouse + REL axes)
```

The dongle is acting as a gamepad **and** a keyboard **and** a mouse simultaneously — a "PC mode" feature for desktop nav from the controller.

### D-input mode — USB ID `2dc8:6012`

Exposes a single device, gamepad only:

```
N: Name="8BitDo 8BitDo Ultimate 2 Wireless Controller for PC"  → event10 / js0
```

### Why composite mode breaks games

Modern controller-aware games typically rely on a higher-level "gamepad" classification (e.g. SDL2's GameController API, used by SDL2 games and Unity on Linux) rather than reading raw joystick devices directly. With the controller exposed as composite HID — gamepad + keyboard + mouse on the same parent device — the gamepad classification path appears to fail, and the game sees no controller. In D-input mode, with only the gamepad interface present, classification succeeds and the game picks it up normally.

This also explains the Bluetooth path: Bluetooth on this controller uses D-input by default, so the composite-HID problem doesn't appear there.

---

## Fix

Boot the controller into D-input mode each time it's powered on.

1. Power off the controller fully — hold the home/guide button for ~3 seconds.
2. Power on while **holding B**.
3. Verify the dongle is in the correct mode:
   ```bash
   lsusb | grep 8bitdo
   # Expected: ID 2dc8:6012  (D-input mode)
   # Wrong:    ID 2dc8:310b  (default / composite mode)
   ```
4. Optionally verify the kernel side:
   ```bash
   cat /proc/bus/input/devices | awk '/8BitDo/,/^$/'
   # Should show one device, gamepad only — no keyboard or mouse entries
   ```

The mode does **not** persist across power cycles — the controller boots into default/composite mode if powered on without B held.

### Other dongle modes (for reference)

| Boot button held | Mode | Exposure |
|---|---|---|
| (none) | Default / composite | Gamepad + keyboard + mouse — breaks SDL2 games |
| **B** | D-input | Single gamepad device — works |
| **Y** | Switch (HID) | Single gamepad, `hid-nintendo` driver |

---

## Diagnostic dead ends

For future reference, the following did not change behavior:

- Disabling Steam Input per-game (Properties → Controller → "Disable Steam Input")
- Forcing the native Linux binary on a game whose Steam package ships Windows binaries by default (via Compatibility → Steam Linux Runtime 1.0 / Scout) — relevant for some games but unrelated to controller detection
- Toggling Steam's global "Generic Gamepad Configuration Support" / "Xbox Configuration Support"

The diagnosis came from `cat /proc/bus/input/devices`, which made the composite-vs-single device structure visible.

---

## Useful diagnostic commands

```bash
# Identify dongle USB ID and current mode
lsusb | grep 8bitdo

# Inspect kernel input device exposure
cat /proc/bus/input/devices | awk '/8BitDo/,/^$/'

# Confirm raw input is flowing
evtest                       # interactive: pick the 8BitDo event node
jstest /dev/input/js0        # legacy joydev path

# See what process holds the controller open while a game is running
sudo lsof /dev/input/event* /dev/input/js* /dev/hidraw* 2>/dev/null | grep -iE '8bitdo|2dc8'
```

---

## Optional: 8BitDo Ultimate Software V2 under Wine

The official 8BitDo Ultimate Software V2 (Windows) can be launched under Wine for tasks like button remapping, macros, profile management, and stick/trigger configuration. The DPI setting in `winecfg` may need adjustment on high-resolution displays; a virtual desktop was not required in testing.

Firmware updates via Wine were **not** attempted in this setup. USB enumeration interruptions during a firmware flash carry recovery risk, so firmware updates are better performed from a Windows environment if needed.
