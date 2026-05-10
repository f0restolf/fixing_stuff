# fixing_stuff

Personal knowledge base of fixes, workarounds, and notes for problems I've run into — mostly with old, unsupported, or niche hardware on Linux (Nobara / Fedora). When I solve something, I dump notes here so the next person hitting the same error message gets a result.

Each subdirectory covers one problem: what it was, what didn't work, what did.

---

## Index

### [8bitdo-ultimate-2-wireless](./8bitdo-ultimate-2-wireless) — 8BitDo Ultimate 2 Wireless: 2.4GHz dongle ignored by games

The controller works fine over Bluetooth and Steam Big Picture sees it, but games don't pick it up. Root cause: the dongle's default firmware mode exposes the controller as a composite HID device (gamepad + keyboard + mouse), which breaks SDL2's GameController classification. Fix: hold **B** while powering on to boot the controller into D-input mode.

**Stack:** 8BitDo Ultimate 2 Wireless + 2.4GHz dongle, Nobara 43, kernel 6.17.x, SDL2-based games, Unity-on-Proton.

**Likely search hits:** 8BitDo Ultimate 2 Wireless not working in games, dongle USB ID `2dc8:310b` vs `2dc8:6012`, composite HID gamepad, SDL2 `GameController` classification, Steam Input dongle Linux.

### [hp560sfp-firmware-update](./hp560sfp-firmware-update) — Flashing HP 560SFP+ firmware on non-ProLiant hardware

HPE's firmware update tool refuses to run on consumer motherboards, but the firmware files inside the package fully support the 560SFP+. Ancient EEPROM firmware on used cards (e.g. `0x800007C7`) can hang POST/GRUB when SFP+ modules are linked at boot. Fix: force-install the RPM, symlink `libpci.so.3`, unbind the `ixgbe` driver, run `.setup` interactively (no flags) — this bypasses the HPSUM platform check.

**Stack:** HP 560SFP+ (Intel 82599ES, PCI `8086:10fb` / `103c:17d3`), `ixgbe`, Linux x86_64.

**Likely search hits:** HP 560SFP+ firmware update Linux, HPE `firmware-nic-intel-1.27.30`, `Possible execution of smart component from an older unsupported HPSUM release`, NIC firmware flash without ProLiant, `Adapter initialization failed` during NVM update.

### [ixgbe-third-party-sfp](./ixgbe-third-party-sfp) — Third-party SFP+ modules on Intel `ixgbe` NICs

Three loosely related issues running non-Intel SFP+ transceivers on `ixgbe`-driven NICs (Intel X520-DA1, HP 560SFP+, etc.): (1) `ixgbe` rejects unrecognized SFP+ modules by default — fix with `allow_unsupported_sfp=1`; (2) some copper RJ45 transceivers (e.g. RealHD) spoof optical DDM data, so `ethtool -m` reports nonsense laser/power values, while honest modules like the MikroTik S+RJ10 report `-inf dBm` zeros — only temperature and voltage are reliable across modules; (3) RJ45 SFP+ modules run hot in dead-air bracket pockets and need airflow. For temperature monitoring see the [CC-SFP-module-sensor](https://github.com/f0restolf/CC-SFP-module-sensor) daemon — module-agnostic, writes hwmon-format files for CoolerControl.

**Stack:** Intel X520-DA1 / HP 560SFP+ (Intel 82599ES) or any `ixgbe`-driven NIC, MikroTik S+RJ10 / RealHD SFP+ → RJ45 transceivers, Nobara / Fedora.

**Likely search hits:** `ixgbe allow_unsupported_sfp`, MikroTik S+RJ10, RealHD SFP+ RJ45 10GBase-T, SFP+ DDM spoofed laser power, copper SFP+ thermal throttle, CoolerControl SFP+ custom file sensor, `ethtool -m` bogus optical readings.

---

## Why this repo exists

Hardware-specific Linux problems tend to have their solutions buried in a forum post, a kernel mailing list thread, or a five-year-old GitHub issue. This is my own breadcrumb trail back to the working answer, made public so others can find it too.

Notes are written from a Nobara Linux 43 (Fedora-based) workstation but most fixes apply to any modern Fedora / RHEL-family system, and many to Linux in general.

## Contributing

Issues and PRs welcome — corrections, better workarounds, or related fixes to add. If you hit one of these problems on a different distro / kernel / hardware revision and the fix needed tweaking, a note about that is useful.

## License

[MIT](./LICENSE) — use, modify, redistribute freely. No warranty.
