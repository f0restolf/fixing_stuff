# Flashing HP 560SFP+ (82599ES) Firmware on Non-ProLiant Hardware — Linux

## The Problem

The **HP Ethernet 10Gb 2-port 560SFP+ Adapter** (PCI ID `8086:10fb`, subsystem `103c:17d3`) ships with ancient EEPROM firmware on secondhand cards. Old firmware (e.g. `0x800007C7`) can cause **system freezes during POST/GRUB** when SFP+ transceivers with an active link are plugged in at boot time. The card works perfectly once the OS has loaded.

HPE's official firmware update tool (`hpsetup` / `.setup`) is designed for ProLiant servers and **refuses to run on consumer motherboards**, even though the firmware files inside the package fully support the 560SFP+.

This guide documents how to bypass the platform restriction and flash the firmware on any Linux x86_64 system.

## What You Need

- HP Ethernet 10Gb 2-port 560SFP+ Adapter (HP P/N: 665249-B21, spare: 669279-001, assembly: 665247-001)
- Linux x86_64 (tested on Nobara 43 / Fedora 43 base, kernel 6.19.x)
- Root access
- The `pciutils` package (`lspci`, `libpci.so.3`)

## Quick Reference

| Item | Value |
|------|-------|
| NIC Chipset | Intel 82599ES |
| Linux Driver | `ixgbe` (in-kernel) |
| PCI Vendor:Device | `8086:10fb` |
| PCI Subsystem | `103c:17d3` |
| Old EEPROM (typical used card) | `0x800007C7` |
| Target EEPROM | `0x800009E1` |
| Old Option ROM | `1.399.0` |
| Target Option ROM | `1.3299.0` |

## Step 1 — Download the HPE Firmware Package

Download **HPE Intel Online Firmware Upgrade Utility Linux x86_64** version **1.27.30** from HPE's support site:

[https://support.hpe.com/connect/s/softwaredetails?collectionId=MTX-d8c86b5c3ce7444d](https://support.hpe.com/connect/s/softwaredetails?collectionId=MTX-d8c86b5c3ce7444d)

You need version **1.27.30 or earlier** — this is the last version that includes 560SFP+ firmware files. Version 1.37.0+ dropped support for the 560SFP+.

The file is: `firmware-nic-intel-1.27.30-1.1.x86_64.rpm`

> **Note:** HPE may also offer the package via a `curl` copy button on the download page.

## Step 2 — Install the Package

The RPM may fail digest verification on newer Fedora/RHEL systems. Force install:

```bash
sudo rpm -ivh --nodigest --nosignature firmware-nic-intel-1.27.30-1.1.x86_64.rpm
```

Verify installation:

```bash
rpm -ql firmware-nic-intel-1.27.30-1.1.x86_64 | head -5
```

The package installs to:
```
/usr/lib/x86_64-linux-gnu/firmware-nic-intel-1.27.30-1.1/
```

## Step 3 — Fix the Missing `libpci.so.3` Dependency

The bundled `.setup` binary needs `libpci.so.3` but doesn't ship it. Symlink your system's copy:

```bash
# Find your system's libpci
find /usr -name "libpci.so.3*" -not -type l 2>/dev/null

# Symlink it into the package's deps directory
sudo ln -s /usr/lib64/libpci.so.3 \
  /usr/lib/x86_64-linux-gnu/firmware-nic-intel-1.27.30-1.1/deps/libs/libpci.so.3
```

Adjust the source path if your distro puts `libpci.so.3` elsewhere. Install `pciutils-libs` if it's missing:

```bash
sudo dnf install -y pciutils-libs
```

## Step 4 — Unbind the ixgbe Driver

**This is the critical step.** The NVM update library cannot access the card's EEPROM while the kernel driver holds the device. Unbind both ports:

```bash
sudo bash -c 'echo 0000:03:00.0 > /sys/bus/pci/drivers/ixgbe/unbind'
sudo bash -c 'echo 0000:03:00.1 > /sys/bus/pci/drivers/ixgbe/unbind'
```

> **Replace `03:00.0` / `03:00.1`** with your card's actual PCI addresses. Find them with:
> ```bash
> lspci -nn | grep 82599
> ```

Verify the driver is unbound:

```bash
lspci -nnk -s 03:00.0 | grep "driver"
# Should return nothing (no "Kernel driver in use" line)
```

## Step 5 — Flash the Firmware

Run `.setup` in **interactive mode with no flags** from the package directory:

```bash
cd /usr/lib/x86_64-linux-gnu/firmware-nic-intel-1.27.30-1.1
sudo LD_LIBRARY_PATH=./deps/libs ./.setup
```

The tool will:
1. Detect the 560SFP+ adapter
2. Show current vs. target firmware versions
3. Ask for confirmation for each component (EPROM and ROM)

Answer **y** to both prompts:

```
Found HPE Ethernet 10Gb 2-port 560SFP+ Adapter MAC: XXXXXXXXXXXX
Do you want to update the following firmware on 8086.10FB.103C.17D3 :
EPROM   0.0.800007C7 to 0.0.800009E1 - y/n/q: y
ROM   1.399.0 to 1.3299.0 - y/n/q: y
```

Wait for completion:

```
Firmware (EPROM) upgrade on MAC XXXXXXXXXXXX SUCCESSFUL
Firmware (ROM) upgrade on MAC XXXXXXXXXXXX SUCCESSFUL
NIC firmware update completed successfully.
```

## Step 6 — Rebind the Driver and Reboot

```bash
sudo bash -c 'echo 0000:03:00.0 > /sys/bus/pci/drivers/ixgbe/bind'
sudo bash -c 'echo 0000:03:00.1 > /sys/bus/pci/drivers/ixgbe/bind'
```

Reboot for the new firmware to take effect:

```bash
sudo reboot
```

## Step 7 — Verify

After reboot, confirm the new firmware:

```bash
sudo ethtool -i enp3s0f0 | grep firmware
# Should show: firmware-version: 0x800009e1
```

## Cleanup

```bash
sudo rpm -e firmware-nic-intel-1.27.30-1.1.x86_64
```

## Why This Works

HPE's `.setup` binary has two code paths:

1. **Flags like `-u` / `--usexml`** — these check for HPSUM (HP Smart Update Manager) and validate the server platform. They fail on non-ProLiant hardware with "Possible execution of smart component from an older unsupported HPSUM release."

2. **Interactive mode (no flags)** — this bypasses the HPSUM/platform check entirely and directly uses `libnvmupdatelinux.so` to flash the NVM. It only requires that the NIC is detectable on the PCI bus and that no kernel driver is holding the device.

The platform check in interactive mode only validates the *NIC*, not the *server*. As long as the PCI subsystem ID (`103c:17d3`) matches an entry in `hpnvmupdate.cfg`, the flash proceeds.

## Why the Driver Must Be Unbound

When the `ixgbe` kernel driver is bound to the card, it holds exclusive access to the NIC's resources. The NVM update library (`libnvmupdatelinux.so`) needs raw access to the card's EEPROM to read and write firmware. With the driver bound, the library fails with:

```
Adapter initialization failed. Attempt to update the firmware will fail,
so update operation will be skipped.
```

Unbinding the driver releases the PCI device so the library can access it directly.

## Troubleshooting

### "Can't find supported devices in the system!"

This can mean:
- **`libpci.so.3` is missing** — see Step 3. Verify with:
  ```bash
  ls -la /usr/lib/x86_64-linux-gnu/firmware-nic-intel-1.27.30-1.1/deps/libs/libpci.so.3
  ```
- **Driver is still bound** — see Step 4. The tool may detect the card but fail to initialize it.
- **Wrong package version** — versions newer than 1.27.30 dropped 560SFP+ support.

### "Component/Platform architecture mismatch"

You downloaded the 32-bit package. Make sure you have the `x86_64` RPM.

### "Adapter initialization failed" in discovery XML

The `ixgbe` driver is still bound. Unbind it per Step 4.

### Tool shows help text instead of flashing

You're passing flags that trigger the HPSUM code path. Run `.setup` with **no arguments** for interactive mode.

## Confirmed Working Cards

| Card | PCI Subsystem | Firmware Before | Firmware After |
|------|---------------|-----------------|----------------|
| HP 560SFP+ (665249-B21) | `103c:17d3` | `0x800007C7` | `0x800009E1` |

## Confirmed Working Platforms

| Motherboard | CPU | OS |
|-------------|-----|-----|
| ASRock X570 Taichi | Ryzen 7 5800X | Nobara Linux 43 (Fedora 43 base, kernel 6.19.x) |

## Related Cards

This procedure may work for other HP OEM Intel 82599ES cards. The `hpnvmupdate.cfg` file in the package lists firmware for:

- HP Ethernet 10Gb 2-port 560SFP+ Adapter (`103c:17d3`)
- HP Ethernet 10Gb 2-port 560FLR-SFP+ Adapter
- HP Ethernet 10Gb 2-port 560M Adapter

## License

This document describes a procedure for using commercially available firmware tools. No proprietary code is distributed. The firmware and tools are property of Hewlett Packard Enterprise / Intel Corporation. Download them from [HPE Support Center](https://support.hpe.com).
