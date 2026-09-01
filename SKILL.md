---
name: linux-phone-porting
description: Use for every hardware bring-up / debug session when porting mainline Linux to a phone. Enforces evidence-first debugging - capture device logs, then research across mainline, the vendor kernel, postmarketOS, Halium/UBports, Mobian and NixOS - before writing or flashing any fix. Requires an already-unlocked bootloader; locked devices are out of scope.
---

# Linux Phone Porting

On a phone under mainline bring-up, a debug iteration costs a build plus a flash plus a boot (5-10 min), and blind fixes get refuted far more often than they land. Follow this order strictly.

## 0. Set up the port

Do this once, before the first flash. Two things make it phase 0 rather than paperwork: the stock system is the highest-authority research source you will ever have for this board, and parts of it stop being readable the moment you start overwriting partitions.

Establish the target first, because every later answer depends on it:

- **Exact device identity** — marketing name, codename, SoC, and the regional or storage variant. Sibling variants of the same marketing name routinely differ in panel, touch controller, or modem, and a fix researched against the wrong variant looks plausible and fails.
- **Target distro and init** — postmarketOS, Mobian, NixOS, plain Debian, something else. This decides packaging, image layout, and which of the phase 2 sources is closest to home.
- **Boot topology** — A/B or non-A/B, slot layout, and bootloader lock state. **Unlocked bootloaders only.** If the bootloader is locked, stop: this skill does not unlock bootloaders and does not provide unlocking steps — point the user at their OEM's own instructions and resume once the device is unlocked.
- **Recovery** — whether a custom recovery is installed. Not a requirement: a convenience for backing up and restoring.

Then take a full backup of every partition **except userdata**:

- Skip userdata: it is the bulk of the space, it is the user's data, and it is the one partition you can afford to lose.
- **Back up the device-unique partitions and never publish them.** Modem NV/EFS, persist, and their calibration siblings carry the IMEI, radio calibration, and sensor trim for this individual handset. They are not reproducible from any image on the internet, and a port that corrupts them leaves a phone that cannot register on a network. Treat them as both critical and private.
- Hash every image and record the hashes with the partition table alongside them. Later you will need to answer "is what is on the device still what I backed up", and only a hash answers it.
- Verify the backup is actually restorable before the first risky flash, not after one.

What the backup is worth as research, beyond recovery:

- **The stock DTB** — ground truth for this board's device tree, and the tiebreaker whenever the sibling SoC's dtsi disagrees with it.
- **Firmware blobs and their load order** — what the vendor stack actually ships and where it expects to find it.
- **The vendor kernel cmdline and boot image layout** — offsets, header version, and the arguments the stock kernel was given.
- **The exact kernel version string** — `uname -a` or `/proc/version` from the stock ROM, while it can still be read (also unpackable later from the backed-up boot image). It is the fingerprint that selects the right GPL-published OEM source release in phase 2; vendors ship several builds and they differ.
- **Vendor configs** — sensor, modem, and HAL configuration describing interfaces the mainline driver will have to satisfy.

Finally, set the working conventions the rest of the session assumes: a host-side directory that accumulates one subdirectory per boot for logs and pstore, and a note of which recovery path is known to work. If a backup already exists from an earlier attempt, confirm it covers these partitions and that its hashes still verify, rather than assuming it does.

## 1. Gather evidence from the live device first

Capture the failing run's full dmesg before changing anything. Enable the relevant debug knobs _before_ reproducing (e.g. `echo 0xffffffff > /sys/module/<driver>/parameters/debug_mask`), because a second reproduction costs another boot.

Evidence channels, in rough order of reliability:

- **Live shell over the network or USB.** Cheapest channel; use it for anything the device survives.
- **pstore.** Once systemd is up, harvest from `/var/lib/systemd/pstore/`, **not** `/sys/fs/pstore/` — `systemd-pstore.service` _moves_ every record out early in boot, so an empty `/sys/fs/pstore` is not evidence that no crash was recorded. The console region is typically a single slot: copy it to the host before doing anything else, or the next crash overwrites it in the archive too.
- **ramoops.** Treat as lossy. DRAM charge retention decays across the unpowered interval — measured at roughly 6.5 % of bits on one device, enough to fail ECC on the zone headers and lose a record entirely. Grep records _fuzzily_. What rots the region is time spent unpowered, not time spent spamming before you cut power, so let a wedged device log for as long as the console zone can hold, then power-cycle and power back on promptly. Check the zone size before assuming you must hurry: a 4 MiB console zone absorbs hours of watchdog spam before it wraps.
- **Kernel-level USB ACM console** (`CONFIG_U_SERIAL_CONSOLE=y` plus `console=ttyGS0` on the cmdline). The only channel that survives a wedge, and worth its one-time flash cost early in a port. A _userspace_ ACM getty does not survive one — it needs a schedulable CPU, and yields zero bytes while still enumerating fine over USB. Both halves are required: the config symbol alone leaves the facility inert.

Three rules that repeatedly turn inference into measurement:

- **Confirm what is actually flashed by reading the partition back**, never from notes and never from a build log. Read with `iflag=direct` so the read itself does not perturb whatever you are debugging. Beware size-only comparisons — boot images are padded to a fixed size, so equal length across two different images proves nothing. Hash the payload.
- **Reading a debug interface can manufacture the symptom.** DRM/GPU crash-state nodes are the classic case: sampling one while the GPU is actively submitting synthesises fault and recovery lines that never occurred (measured: five reads, five fault/recover pairs, versus none unobserved). Before trusting a log line, check whether your own observation produced it.
- **Record the rules you withdraw.** Bring-up accumulates folklore fast, and a rule that was right for one hardware revision or one buffer size becomes actively misleading after it changes. Keep the overturned version next to the current one with what changed, so the next session knows which advice has been stress-tested.

Host-side gotchas worth pre-empting:

- `timeout N ssh <device> '<cmd>'` kills the local client only; the remote process keeps running. Follow up with an explicit kill.
- `pgrep -f` and `pkill -f` run over that same ssh path **self-match** their own remote shell. Filter the matcher out, or match by PID.
- `find` needs `-L` when the start path is a symlink, or it returns silently empty.
- Any wrapper or hook that rewrites command output — token filters, pagers, formatters — will corrupt evidence, sometimes by inverting a test rather than obviously breaking it. Use the raw command for anything you will commit as a finding.

## 2. Research before implementing — all of these sources

Research always precedes implementation. What it consumes depends on what you are doing:

- **Investigating a failure** — the phase 1 evidence. Capture it first; researching a symptom you have not actually read is guessing with citations.
- **Adding a capability** — a driver, a peripheral, a subsystem that has never come up — the phase 0 artefacts are the device data. The stock DTB, the vendor configs and the firmware layout describe how this board expects the thing to be driven, and they are as authoritative here as a crash log is there.

Either way, **before** writing anything:

1. **Reason about the subsystem** — identify which driver, binding, or firmware interface owns the failure, or would own the new capability. The skills available to you are listed in your session context: scan that list now, and if it has a kernel-development or crash-analysis skill, read it before continuing — its register-level and binding detail is exactly what this step needs. Reason it through directly only when none is listed.
2. **Look up userspace libs and tools involved** — current docs, not recalled API. If a documentation-retrieval skill is in that same session list, run it rather than recalling signatures from memory.
3. **Web research across all six source families** (parallel subagents work well here; if a web-research skill is in that list, prefer it for the caching, since sessions re-read the same pages):
   - **Mainline** — lore.kernel.org, the driver source, DT bindings, and the dtsi of the closest sibling SoC (same family, or same RPM generation).
   - **Downstream vendor kernel — the GPL-published OEM source.** GPLv2 obliges Android OEMs to publish the kernel sources they shipped, so an exact release for this device usually exists: the OEM's open-source portal (Samsung, Sony, Xiaomi, Fairphone, OnePlus/OPPO and Motorola all run one), their GitHub/GitLab mirrors, and community archives such as XDA when a portal link has died. Select the tarball whose version string matches `/proc/version` from phase 0 — vendors publish several, and they differ. Compliance quality varies, so treat the tree as an oracle to mine — the board dts and defconfig for this exact device, and the out-of-tree vendor drivers (touch, panel, charger, modem glue) with their register sequences and firmware handshakes — rather than assuming it builds. When no exact release surfaces, a sibling device's release usually still carries the SoC dtsi. Better still: **the stock DTB pulled off the device itself** (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`). That is ground truth for _this_ board and has corrected wrong sibling-SoC guesses more than once — including a SMMU stream ID that the sibling SoC got wrong and the stock DTB got right.
   - **postmarketOS** — pmaports device packages and APKBUILDs, merge requests, wiki device pages, and the SoC-mainline tree.
   - **Halium / UBports** — `halium/android_device_*` trees, `hybris-boot` configs, and UBports device ports. Halium runs the Android vendor stack rather than mainline, which makes it the best source for how the _vendor firmware_ expects to be driven: firmware blob paths and load order, the property and socket interfaces daemons expect, sensor and modem HAL configs. When a mainline driver probes but the firmware handshake fails — or when you need to know what handshake a not-yet-supported peripheral expects — this is usually where the missing piece is written down.
   - **Mobian** — the Debian device repos.
   - **NixOS** — mobile-nixos patterns and nixpkgs packaging.

**Thin results are the normal case, not a blocker.** Most of what a port needs has never been done on this board, and often not on this SoC at all. Six sources reporting nothing means nobody has published the work — it never means the work should stop or be handed back. When precedent is absent, the answers are still in the primary artefacts: the out-of-tree vendor driver that already drives the peripheral (register sequences, firmware handshake, GPIO and regulator wiring), the stock DTB's own node for it, the vendor HAL config naming what userspace expects, and the closest mainline driver for the same device class as a structural template. Read those, derive the design, and write the missing piece — a DT node, a quirk, a small driver. Authoring that piece is the ordinary terminal state of a bring-up, not an exception to be justified; the phase 0 backup and the one-variable-per-flash rule exist precisely so that a wrong derivation costs one boot and one ledger line, the same as any other refutation.

## 3. Implement only with a promising, evidence-backed hypothesis

- **One variable per flash.** State the hypothesis before flashing.
- Prefer runtime tests over reflashes: push files with a tar pipe, unbind and rebind drivers, `insmod` a module. Seconds instead of minutes.
- Device-tree-only changes are cheap — build just the DTB when the toolchain allows it.
- **After 3 refuted fixes: stop guessing, not stop working.** Three misses from the same model of the failure mean the model is wrong, and a fourth tweak from it will miss too. Return to step 2 with a wider scope — a different layer of the stack, or the primary artefacts directly where precedent is absent — and come back with findings stacked, as in the worked example below.

A worked shape, from a Wi-Fi bring-up that took this path: five blind device-tree tweaks all failed. The fix was five separate findings stacked, every one of them from research rather than guessing — a trustzone memory mode, vendor firmware paths, a host-capability quirk, the SMMU stream ID off the stock DTB, and one property that mainline had made obsolete and needed dropping. No single tweak would have got there.
