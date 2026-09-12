---
description: Establish the target and take the pre-flash backup that later research depends on (phase 0 of linux-phone-porting)
argument-hint: "[device, or path to an existing backup]"
---

Run phase 0 of the `linux-phone-porting` skill for: $ARGUMENTS

This runs once per port, before the first flash; re-check identity and capabilities when the device or transport changes. Do not flash anything during it.

**Establish the target.** Ask the user for anything not already known — do not infer these from a codename alone:

1. Exact device identity: marketing name, internal codenames, OEM/ODM, SoC, regional or storage variant, and EVT/DVT/PVT revision where known. Reconcile labels, bootloader identifiers, Android properties and stock DTB; unresolved contradictions block choosing images. Sibling and prototype variants can differ in panel, touch controller and modem.
2. Target distro and init system.
3. Target userspace shell: which desktop environment or shell the user wants — never assume a default. If the target OS ships its own shell (a UBports port carries its own, for example), strongly recommend sticking with it.
4. Boot topology: actual slot and bank layout, active boot target, and current bootloader lock state. Vendor banked A/B is not proof of Android seamless A/B support. Record unavailable lock fields as unknown, not unlocked.
5. Whether a backup already exists. Verify origin, acquisition method, coverage and hashes separately: a trustworthy download is not a device backup, and matching hashes prove integrity, not completeness or restorability. For prototype hardware, a retail ROM is not a substitute for its own backup.
6. Whether a custom recovery is installed. Not a requirement — a convenience for backing up and restoring.

**Unlocked bootloader required.** Confirm the unlocked state before proceeding to any write-capable workflow; an unknown lock state does not pass this prerequisite. If the bootloader is locked, stop: do not unlock it and do not provide unlocking steps — point the user at their OEM's own instructions and resume once the device is unlocked.

**Take the backup** — every partition except userdata:

- Skip userdata: bulk, private, and the one partition that can be lost.
- Back up modem NV/EFS, persist and their calibration siblings, and flag them to the user as device-unique: they carry IMEI, radio calibration and sensor trim for this handset, cannot be re-downloaded, and must never be published.
- Hash every image; save the hashes and the partition table next to them.
- Confirm the backup is restorable before any risky flash. Prototype-specific boot and calibration data must be secured first; a same-SoC recovery image or loader is only a rescue candidate until compatibility and a recovery path are demonstrated on this target.

**Check the device's health before trusting it.** Read the storage's own wear/lifetime report (UFS life-time percentage, eMMC health register — whatever this storage exposes) and record the raw values and their interpretation. An unavailable field means unknown health. Measured on one device: I/O errors initially blamed on storage cleared after another stock-ROM flash while its wear report indicated healthy lifetime. This separates evidence of wear from software-state failure; it does not certify the whole storage device or the rest of the hardware.

**Inventory and harvest the stock system while it is present.** Rooted stock beats a ROM image as a source: a backup preserves bytes, root keeps the stock system running and readable. Assume the user roots the device themselves; like bootloader unlocking, do not provide rooting steps — resume once root is available. Then:

- Inventory every component the stock system will name — panel, touch controller, sensor set, camera sensors, modem and its RF configuration, WLAN/BT chip, charger and fuel gauge, audio path — and compare against the official spec sheets for this variant: invoke the `find-docs` skill for each component's documentation; if it is not installed, use plain web search. Mark each claim as declared, enumerated, driver-bound, or exercised with its observed result; none automatically implies the next. This catches "researched the wrong variant" without prematurely qualifying untested hardware.
- From rooted stock: the property dump (`getprop`), the stock kernel config (`/proc/config.gz`), the mounted vendor/odm trees, HAL and sensor configs, calibration artefacts, and the factory field-test modes.
- Ask whether recent community custom ROMs exist for the device and record the newest as an optional, up-to-date development source. Measured: the most responsive OS ever run on one device was an unofficial recent-Android custom build, not the stock ROM — and its boot image shares the stock downstream lineage, so its DTB doubles as an independent cross-check.
- Note that a bootable Android — stock or rooted custom — stays available as a data-gathering channel for the whole port, not just phase 0: a working vendor driver's probe and firmware sequence is read live next to a mainline failure. Exercising hardware under Android supports that path under those conditions, not blanket hardware health or mainline support; cross-check before transferring the conclusion.

**Prove the control channels before choosing a build strategy.** Record which channels work, their explicit target selectors, and their demonstrated capabilities:

- Exercise host `adb` and `fastboot` against this device. Pin every command to the verified serial or transport; enumeration alone is not proof of boot, readback, fetch or flashing support.
- Re-enumerate after every mode change and reconcile the new identifier with the target; do not silently select the first device.
- Prove the boot and fetch/readback operations the proposed workflow needs before building for them. Host-tool help and same-SoC precedent do not establish bootloader support. If a capability is absent, choose a demonstrated alternative without a speculative write.
- Exercise at least one post-boot control channel into the target Linux: USB-gadget Ethernet or a serial ACM console, ssh over network or over USB. Record its address and observed readiness rather than merely the installed tools.

**Lock the project to this ruleset.** Before finishing, write an explicit rule into the project's agent context files — `CLAUDE.md`, `AGENTS.md`, and any equivalent the project carries — stating that **all work touching this device goes through the `linux-phone-porting` skill's ruleset, and no action on the device is taken outside it**: every change researches per phase 2, gates per phase 3 (pre-build battery, edit precision, operator-hands verification), and flashes only through the gates above. The port survives on these rules being load-bearing, not advisory — a session that "just quickly" flashes outside the ruleset is how a backup-only-recoverable mistake happens.

**Mine it as a research source**, and tell the user what was found:

- Extract the stock DTB (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`).
- Inventory the firmware blobs and their load order.
- Record the vendor kernel cmdline and the boot image layout: offsets, header version, page size.
- Record the exact kernel version string (`uname -a`, `/proc/version`) — the fingerprint that selects the right GPL-published OEM source release in the research phase.
- Note vendor sensor, modem and HAL configs describing interfaces mainline will have to satisfy.

**Set the conventions**:

- A recovery path that has been shown to work.
- **Project layout:** for multiple targets, use the optional reference layout `kernel/<version>`, `soc/<vendor-soc>`, `os/<distro>`, and `devices/<model>`; a single-device project may stay flat. Keep generic kernel, SoC and OS bases free of device imports; compose device choices at the device layer. Keep artifacts and logs per device, not in a shared ambiguous bucket.
- Everything large, private, or device-derived belongs under that device's gitignored `artifacts/`, never committed, classified as `private/` (device-unique, never leaves the machine; partition backups and the userdata exclusion note), `firmware-harvest/` (blobs pending redaction), `android/` (reproducible stock-ROM packages and rooted captures), `debug-evidence/` (irreplaceable captures), or `reference/` (reading copies). Publishable firmware goes to a sibling `firmware-publishable/` repository with its own git history, derived only after redaction. Per-device `logs/` takes one subdirectory per boot or deploy plus `LATEST-*` links. An artifact manifest maps paths to their class and records origin, acquisition method, coverage and hashes.

Report: the reconciled target identity and unresolved fields, backup provenance, coverage, integrity and restore evidence separately, the research artefacts extracted, hardware evidence levels, demonstrated transport capabilities and recovery limitations, and whether Android remains available as a data-gathering boot.
