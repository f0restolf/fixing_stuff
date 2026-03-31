# HP 560SFP+ with RealHD SFP+ RJ45 Transceivers

## The hardware

- **NIC:** HP 560SFP+ (Intel 82599ES, `ixgbe` driver)
- **Transceivers:** RealHD SFP+ to RJ45 10GBase-T

## The problems

### 1. ixgbe rejects the transceivers

The `ixgbe` driver blocks third-party SFP+ modules by default. However, this is not necessarily always the case and some modules may be automatically loaded, mine were, YMMV.

**Fix:** Load ixgbe with `allow_unsupported_sfp=1`:

```bash
echo "options ixgbe allow_unsupported_sfp=1" | sudo tee /etc/modprobe.d/ixgbe.conf
sudo dracut --force   # rebuild initramfs (Fedora/Nobara)
```

Reboot for it to take effect. Verify after boot:

```bash
cat /sys/module/ixgbe/parameters/allow_unsupported_sfp
# Should print: 1
```

### 2. Transceivers spoof optical module identity

The RealHD modules are copper RJ45 transceivers but they report themselves as optical SFP+ (LC connector, 850nm laser, SR optics). This means `ethtool -m` dumps a wall of garbage: fake laser bias current, fake optical power, fake receiver power. Fuhgeddaboudit

The only useful data in the DDM output:

| Field | Real? | Notes |
|-------|-------|-------|
| Module temperature | **Yes** | Actual transceiver die temp |
| Module voltage | **Yes** | Actual 3.3V supply rail |
| Laser bias current | No | Spoofed, meaningless |
| Laser output power | No | Spoofed, meaningless |
| Receiver signal power | No | Spoofed, meaningless |

Thresholds for temperature and voltage are also real (95°C alarm, 85°C warning, 3.0-3.6V voltage range).

### 3. Transceivers run hot

RJ45 SFP+ modules pull 2-3W each. In a dead air pocket behind a PCIe bracket, they can thermal throttle or shut down. A 40mm fan pointed at the SFP+ cage should solve this completely. For piece of mind or placebo (if its working, who cares?) I slapped one of those lil heatsinks on there and it might not do anything but I love my fins (the best part of heat transfer).

## Temperature monitoring

The spoofed DDM output made my previous monitoring tool attempts useless — they choked on the fake data or reported nonsense. I am a grease monkey by nature not a CS major.

**[sfp-monitor](https://github.com/f0restolf/sfp-monitor)** is a lightweight C daemon that polls `ethtool -m`, extracts only the real temperature and voltage readings, and writes them as hwmon-format files for CoolerControl (CC) to consume. It runs as a systemd service and outputs to `/run/sfp-monitor/` (tmpfs). To actually implement it in CC you will have to create a custom sensor and use the from file option, point it to the files created ofc. The voltage I havent really used but hey, its there!

## HP NIC naming cheat sheet

Easily confused models:

| Model | Chipset | Driver | Verdict |
|-------|---------|--------|---------|
| HP 530SFP+ | Broadcom BCM57810S | `bnx2x` | Avoid |
| HP 530T | Broadcom BCM57810S | `bnx2x` | Avoid |
| HP 560SFP+ | **Intel 82599ES** | `ixgbe` | Good |
| HP 561T | Intel X540 | `ixgbe` | Good |
| HP 560FLB | Intel 82599ES | `ixgbe` | Wrong form factor (FlexibleLOM, not PCIe) |
