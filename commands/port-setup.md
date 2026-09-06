---
description: Establish the target and take the pre-flash backup that later research depends on (phase 0 of linux-phone-porting)
argument-hint: "[device, or path to an existing backup]"
---

Run phase 0 of the `linux-phone-porting` skill for: $ARGUMENTS

This runs once per port, before the first flash. Do not flash anything during it.

**Establish the target.** Ask the user for anything not already known — do not infer these from a codename alone:

1. Exact device identity: marketing name, codename, SoC, and regional or storage variant. Note that sibling variants differ in panel, touch controller and modem, so record which one this is.
2. Target distro and init system.
3. Target userspace shell: which desktop environment or shell the user wants — never assume a default. If the target OS ships its own shell (a UBports port carries its own, for example), strongly recommend sticking with it.
4. Boot topology: A/B or non-A/B, slot layout, current bootloader lock state.
5. Whether a backup already exists. If the user points to one, verify its coverage and re-check its hashes rather than trusting the directory name.
6. Whether a custom recovery is installed. Not a requirement — a convenience for backing up and restoring.

**Unlocked bootloader required.** If the bootloader is locked, stop: do not unlock it and do not provide unlocking steps — point the user at their OEM's own instructions and resume once the device is unlocked.

**Take the backup** — every partition except userdata:

- Skip userdata: bulk, private, and the one partition that can be lost.
- Back up modem NV/EFS, persist and their calibration siblings, and flag them to the user as device-unique: they carry IMEI, radio calibration and sensor trim for this handset, cannot be re-downloaded, and must never be published.
- Hash every image; save the hashes and the partition table next to them.
- Confirm the backup is restorable before any risky flash.

**Check the device's health before trusting it.** Read the storage's own wear/lifetime report (UFS life-time percentage, eMMC health register — whatever this storage exposes) and record it. Storage is the classic misdiagnosis: I/O errors have been read as a failing drive that the wear percentage proved healthy, with the real fault clearing on one more fastboot flash of the stock ROM. The percentage separates "hardware is dying" from "software state is bad" — they have opposite next steps.

**Inventory and harvest the stock system while it is present.** Rooted stock beats a ROM image as a source: a backup preserves bytes, root keeps the stock system running and readable. Assume the user roots the device themselves; like bootloader unlocking, do not provide rooting steps — resume once root is available. Then:

- Inventory every component the stock system will name — panel, touch controller, sensor set, camera sensors, modem and its RF configuration, WLAN/BT chip, charger and fuel gauge, audio path — and compare against the official spec sheets for this variant: invoke the `find-docs` skill for each component's documentation; if it is not installed, use plain web search. This is what catches "researched the wrong variant" before it costs a flash.
- From rooted stock: the property dump (`getprop`), the stock kernel config (`/proc/config.gz`), the mounted vendor/odm trees, HAL and sensor configs, calibration artefacts, and the factory field-test modes.
- Ask whether recent community custom ROMs exist for the device and record the newest as an optional, up-to-date development source. Measured: the most responsive OS ever run on one device was an unofficial recent-Android custom build, not the stock ROM — and its boot image shares the stock downstream lineage, so its DTB doubles as an independent cross-check.
- Note that a bootable Android — stock or rooted custom — stays available as a data-gathering channel for the whole port, not just phase 0: a working vendor driver's probe and firmware sequence is read live next to a mainline failure, and hardware mainline cannot yet drive is exercised under Android to prove the silicon is fine. Never conclude anything about mainline behaviour from Android behaviour without a cross-check.

**Prove the control channels.** Before finishing, verify the agent can actually reach the device on every channel it will need later, and record which work with their addresses:

- `adb` and `fastboot` on the host, exercised against this device (`adb devices`, `fastboot devices` — an untested install is not a channel).
- At least one post-boot control channel into the target Linux: USB-gadget Ethernet or a serial ACM console, ssh over network or over USB.

**Lock the project to this ruleset.** Before finishing, write an explicit rule into the project's agent context files — `CLAUDE.md`, `AGENTS.md`, and any equivalent the project carries — stating that **all work touching this device goes through the `linux-phone-porting` skill's ruleset, and no action on the device is taken outside it**: every change researches per phase 2, gates per phase 3 (pre-build battery, edit precision, operator-hands verification), and flashes only through the gates above. The port survives on these rules being load-bearing, not advisory — a session that "just quickly" flashes outside the ruleset is how a backup-only-recoverable mistake happens.

**Mine it as a research source**, and tell the user what was found:

- Extract the stock DTB (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`).
- Inventory the firmware blobs and their load order.
- Record the vendor kernel cmdline and the boot image layout: offsets, header version, page size.
- Record the exact kernel version string (`uname -a`, `/proc/version`) — the fingerprint that selects the right GPL-published OEM source release in the research phase.
- Note vendor sensor, modem and HAL configs describing interfaces mainline will have to satisfy.

**Set the conventions**:

- A recovery path that has been shown to work.
- **Project layout**, modelled on the rules the skill carries: everything large, private, or device-derived under `artifacts/` at the project root — gitignored, never committed — and classified: `private/` (device-unique, never leaves the machine; the partition backup lives here, with a DO-NOT-RESTORE note for userdata), `firmware-harvest/` (blobs pending redaction), `android/` (reproducible stock-ROM packages and rooted captures), `debug-evidence/` (irreplaceable captures), `reference/` (reading copies). Publishable firmware goes to a sibling `firmware-publishable/` repository with its own git history, derived only after redaction. `logs/` takes one subdirectory per boot or deploy plus `LATEST-*` links. A README in `artifacts/` maps every path to its class and records hashes.

Report: the target as confirmed by the user, what the backup covers and where it lives, the hashes, the research artefacts extracted from it, the verified control channels, and whether an Android system remains available as a data-gathering boot.
