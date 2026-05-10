# Third-party SFP+ Modules on Intel `ixgbe` NICs

Notes on running non-Intel SFP+ transceivers in Intel `ixgbe`-driven 10GbE NICs (Intel X520-DA1, HP 560SFP+ / 82599ES, etc.) under Linux. Tested on Nobara 43 with HP 560SFP+ and X520-DA1, but the issues are generic to the driver and the SFP+ spec.

## 1. `ixgbe` rejects third-party SFP+ modules

The `ixgbe` driver blocks unrecognized SFP+ modules by default. Some modules slip through depending on what the EEPROM advertises, but anything Intel hasn't blessed is at the driver's mercy. YMMV.

**Fix** — load `ixgbe` with `allow_unsupported_sfp=1`:

```bash
echo "options ixgbe allow_unsupported_sfp=1" | sudo tee /etc/modprobe.d/ixgbe.conf
sudo dracut --force   # rebuild initramfs (Fedora/Nobara)
```

Reboot for it to take effect. Verify:

```bash
cat /sys/module/ixgbe/parameters/allow_unsupported_sfp
# Should print: 1
```

## 2. DDM data sanity — copper SFP+ transceivers can lie

`ethtool -m` reads SFF-8472 Digital Diagnostics Monitoring (DDM) bytes from the module's EEPROM. RJ45 copper transceivers don't have a laser, so optical-related fields are either reported as honest zeros or — on cheaper modules — populated with fake values pretending to be optical. Whether you can trust the output varies a lot by module.

What's real on the two modules I've tested:

| Field                    | RealHD SFP+ → RJ45     | MikroTik S+RJ10                  |
|--------------------------|------------------------|----------------------------------|
| Module temperature       | ✓ real                | ✓ real                          |
| Module voltage           | ✓ real                | ✓ real                          |
| Laser bias current       | ✗ spoofed garbage     | `0.000 mA` (honest zero)         |
| Laser output power       | ✗ spoofed garbage     | `-inf dBm` (honest no-signal)    |
| Receiver signal power    | ✗ spoofed garbage     | `-inf dBm` (honest no-signal)    |
| Alarm/warning thresholds | ✓ real (95/85°C, 3.0–3.6V) | not implemented              |

Temperature and voltage are reliable on both. The optical fields on the RealHD are completely fabricated — `ethtool -m` dumps a wall of plausible-looking laser numbers for a transceiver that has no laser. Fuhgeddaboudit. The MikroTik handles the same situation correctly, reporting honest zeros / `-inf dBm` for the optical fields.

**If you're scripting against `ethtool -m`, only trust temperature and voltage.** Everything else is module-dependent and may be a lie.

## 3. RJ45 SFP+ modules run hot

10GBASE-T transceivers pull 2–3 W each. In a dead air pocket behind a PCIe bracket they can thermal-throttle or shut down. A 40 mm fan pointed at the SFP+ cage solves it completely. For peace of mind / placebo (if it's working, who cares?) I slapped a small heatsink on top — might not do anything but I love my fins (the best part of heat transfer).

## Temperature monitoring → CoolerControl

[CC-SFP-module-sensor](https://github.com/f0restolf/CC-SFP-module-sensor)
is a small module-agnostic C daemon that polls `ethtool -m`, extracts the temperature and voltage readings, and writes them as hwmon-format files (`temp1_input`, `in1_input`) for CoolerControl to consume as a custom file sensor. Runs as a systemd service, outputs to `/run/sfp-monitor/<iface>/` (tmpfs).

To wire it up in CoolerControl: create a custom sensor using the "from file" option and point it at the `temp1_input` file the daemon produces. Voltage is also exposed but I haven't found a real use for it yet.

## HP / Intel 10GbE NIC cheat sheet

Easy ones to confuse when shopping for used 10GbE cards:

| Model           | Chipset             | Driver   | Verdict                         |
|-----------------|---------------------|----------|---------------------------------|
| HP 530SFP+      | Broadcom BCM57810S  | `bnx2x`  | Avoid                           |
| HP 530T         | Broadcom BCM57810S  | `bnx2x`  | Avoid                           |
| HP 560SFP+      | **Intel 82599ES**   | `ixgbe`  | Good                            |
| HP 561T         | Intel X540          | `ixgbe`  | Good                            |
| HP 560FLB       | Intel 82599ES       | `ixgbe`  | FlexibleLOM form factor, not PCIe |
| Intel X520-DA1  | Intel 82599ES       | `ixgbe`  | Good                            |
