---
description: Establish the target and take the pre-flash backup that later research depends on (phase 0 of linux-phone-porting)
argument-hint: "[device, or path to an existing backup]"
---

Run phase 0 of the `linux-phone-porting` skill for: $ARGUMENTS

This runs once per port, before the first flash. Do not flash anything during it.

**Establish the target.** Ask the user for anything not already known — do not infer these from a codename alone:

1. Exact device identity: marketing name, codename, SoC, and regional or storage variant. Note that sibling variants differ in panel, touch controller and modem, so record which one this is.
2. Target distro and init system.
3. Boot topology: A/B or non-A/B, slot layout, current bootloader lock state.
4. Whether a backup already exists. If the user points to one, verify its coverage and re-check its hashes rather than trusting the directory name.
5. Whether a custom recovery is installed. Not a requirement — a convenience for backing up and restoring.

**Unlocked bootloader required.** If the bootloader is locked, stop: do not unlock it and do not provide unlocking steps — point the user at their OEM's own instructions and resume once the device is unlocked.

**Take the backup** — every partition except userdata:

- Skip userdata: bulk, private, and the one partition that can be lost.
- Back up modem NV/EFS, persist and their calibration siblings, and flag them to the user as device-unique: they carry IMEI, radio calibration and sensor trim for this handset, cannot be re-downloaded, and must never be published.
- Hash every image; save the hashes and the partition table next to them.
- Confirm the backup is restorable before any risky flash.

**Mine it as a research source**, and tell the user what was found:

- Extract the stock DTB (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`).
- Inventory the firmware blobs and their load order.
- Record the vendor kernel cmdline and the boot image layout: offsets, header version, page size.
- Record the exact kernel version string (`uname -a`, `/proc/version`) — the fingerprint that selects the right GPL-published OEM source release in the research phase.
- Note vendor sensor, modem and HAL configs describing interfaces mainline will have to satisfy.

**Set the conventions**: a host-side log directory that takes one subdirectory per boot, and a recovery path that has been shown to work.

Report: the target as confirmed by the user, what the backup covers and where it lives, the hashes, and the research artefacts extracted from it.
