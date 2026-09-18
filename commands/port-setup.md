---
description: Establish the target and take the pre-flash backup that later research depends on (phase 0 of linux-phone-porting)
argument-hint: "[device, or path to an existing backup]"
---

Run phase 0 of the `linux-phone-porting` skill for: $ARGUMENTS

This runs once per port, before the first flash; re-check identity and capabilities when the device or transport changes. Do not flash anything during it.

**Establish the target.** Ask the user for anything not already known — do not infer these from a codename alone:

1. Exact device identity: marketing name, internal codenames, OEM/ODM, SoC, regional/storage variant, EVT/DVT/PVT revision where known. Reconcile labels, bootloader identifiers, retail-OS properties (Android build props where present) and the stock board description; unresolved contradictions block choosing images. Sibling/prototype variants differ in panel, touch, modem.
2. Target OS/platform and release, required kernel lineage, startup/service manager, hardware-service interfaces, build/image/update model and evidence/control tools. Verify the target project's requirements; Linux-based does not imply mainline-compatible.
3. Target user interface/session: ask when not already specified; never assume a default. Prefer the target platform's supported interface and session stack.
4. Boot-model class first — retail-unlock, exploit-booted, or firmware-boot — then boot topology: actual slot and bank layout, active boot target, and lock state where the concept exists. The class selects the write gate, the backup route and the board-authority artefact (boot-image DTB, ACPI tables, DT inside the vendor kernel image). Vendor banked A/B is not proof of seamless A/B support. Record unavailable lock fields as unknown, not unlocked.
5. Whether a backup already exists — verify origin, acquisition method, coverage and hashes separately: a trustworthy download is not a device backup; matching hashes prove integrity, not completeness or restorability. For prototype hardware a retail ROM is not a substitute for its own backup.
6. Whether a custom recovery is installed. Not a requirement — a convenience for backing up and restoring.

**Demonstrated boot path required.** Retail-unlock: confirm the unlocked bootloader before any write-capable workflow; an unknown lock state does not pass this prerequisite; if locked, stop — no unlocking steps, point the user at their OEM's own instructions and resume once unlocked. Exploit-booted: require a payload boot path demonstrated on this device; no exploit-landing steps — point at the payload project's own instructions. Firmware-boot: require a demonstrated boot path. In every class the prerequisite is demonstrated capability, never a promised one.

**Take the backup** — every readable partition except userdata; where the class exposes none, back up through the demonstrated route and record the coverage achieved:

- Exclude userdata because it is bulk and private; record the exclusion. This does not authorize wiping it. Explain consequences and preservation options and obtain separate informed authorization before an operation that may destroy it.
- Back up modem NV/EFS, persist and their calibration siblings, and flag them to the user as device-unique: they carry IMEI, radio calibration and sensor trim for this handset, cannot be re-downloaded, and must never be published.
- Apply phase 0's private-calibration rule: validate identity and integrity, use protected runtime access, and keep values out of builds, arguments, logs and public reproductions. Missing calibration must not be replaced with guessed or sibling-device values.
- Hash every image; save the hashes and the partition table next to them.
- Confirm the backup is restorable before any risky flash. Prototype-specific boot and calibration data must be secured first; a same-SoC recovery image or loader is only a rescue candidate until compatibility and a recovery path are demonstrated on this target.

**Check storage health before trusting it.** Read the wear/lifetime report (UFS life-time %, eMMC health register), record raw values and interpretation; unavailable field = unknown health. Measured: I/O errors blamed on storage cleared after another stock-ROM flash while the wear report showed healthy lifetime — separates wear evidence from software-state failure.

**Inventory and harvest while stock is present.** Readable retail beats a ROM image: on Android that is root; other retail OSes have their own readable surfaces — enumerate what this one exposes. Privilege steps are the user's — no rooting or jailbreaking steps; resume when access exists. Then:

- Inventory every component the stock system names — panel, touch, sensors, cameras, modem + RF config, WLAN/BT, charger + fuel gauge, audio — and compare against this variant's official spec sheets: `find-docs` for component docs, `wigolo` for web-found material (cache matters across sessions), else plain web search. Mark each claim declared/enumerated/driver-bound/exercised with its observed result; none implies the next.
- Persist the component–wiring–document inventory per the skill's fields and the project's format: identity evidence separate from functional results; alternatives, unknowns, contradictions explicit. Seed the document index; phase 2's procedure fills gaps. A complete record may honestly contain unknowns.
- From rooted stock: the property dump (`getprop`), the stock kernel config (`/proc/config.gz`), the mounted vendor/odm trees, HAL and sensor configs, calibration artefacts, and the factory field-test modes.
- Ask whether recent community custom ROMs exist; record the newest as an optional development source (measured: the most responsive OS on one device was an unofficial recent-Android build, and its boot image shares the stock downstream lineage, so its DTB cross-checks phase 0). A bootable retail OS stays a data-gathering channel for the whole port — cross-check before transferring conclusions to mainline.

**Prove the control channels before choosing a build strategy.** Record which channels work, their explicit target selectors, and their demonstrated capabilities:

- Discover and exercise the target's available control/recovery tools; `adb` and fastboot are examples where supported, not mandatory tools. Pin every command to the verified serial or transport; enumeration alone is not proof of boot, readback, fetch or flashing support.
- Re-enumerate after every mode change and reconcile the new identifier with the target; do not silently select the first device.
- Prove the boot and fetch/readback operations the proposed workflow needs before building for them. Host-tool help and same-SoC precedent do not establish bootloader support. If a capability is absent, choose a demonstrated alternative without a speculative write.
- Exercise at least one post-boot control channel into target Linux, using its supported mechanism: serial/ACM, USB-gadget Ethernet, ssh or a platform-specific tool. Record its selector/address, demonstrated operations and observed readiness rather than merely the installed tools.

**Lock the project to this ruleset.** Write into the project's agent context files (`CLAUDE.md`, `AGENTS.md`, equivalents): **all work touching this device goes through the skill's ruleset; no device action outside it** — phase 2 research, phase 3 gates, flashes only through the gates above. These rules are load-bearing, not advisory: a session that "just quickly" flashes outside them is how a backup-only-recoverable mistake happens.

**Mine it as a research source**, and tell the user what was found:

- Extract the stock board description for this class: boot-image DTB where the chain carries one (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`); ACPI tables on firmware-boot; the DT inside the vendor kernel image where exploit-booted. Record which one was obtained.
- Inventory the firmware blobs and their load order.
- Record the vendor kernel cmdline and the boot-image layout where present: offsets, header version, page size.
- Record the exact kernel version string (`uname -a`, `/proc/version`) — the fingerprint that selects the right GPL-published OEM source release in the research phase.
- Note vendor sensor, modem and HAL configs describing interfaces mainline will have to satisfy.

**Set the conventions**:

- A recovery path that has been shown to work.
- **Project layout:** for multiple targets, use the optional reference layout `kernel/<version>`, `soc/<vendor-soc>`, `os/<platform>`, and `devices/<model>`; a single-device project may stay flat. This is illustrative, not an instruction to rename existing directories. Keep generic kernel, SoC and OS bases free of device imports; compose device choices at the device layer. Keep artifacts and logs per device, not in a shared ambiguous bucket.
- Everything large, private, or device-derived belongs under that device's gitignored `artifacts/`, never committed, classified as `private/` (device-unique, never leaves the machine; partition backups and the userdata exclusion note), `firmware-harvest/` (blobs pending redaction), `android/` (reproducible stock-ROM packages and rooted captures), `debug-evidence/` (irreplaceable captures), or `reference/` (reading copies). Publishable firmware goes to a sibling `firmware-publishable/` repository with its own git history, derived only after redaction. Per-device `logs/` takes one subdirectory per boot or deploy plus `LATEST-*` links. An artifact manifest maps paths to their class and records origin, acquisition method, coverage and hashes.

Report: the reconciled target identity, OS integration contract, intended interface/session and unresolved fields; backup provenance, coverage, integrity and restore evidence separately; the research artefacts extracted; the persistent inventory and linked document/acquisition records with unresolved gaps; hardware evidence levels; demonstrated transport capabilities and recovery limitations; and whether the retail OS remains available as a data-gathering boot.
