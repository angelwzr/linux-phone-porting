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
- **Target userspace shell** — ask the user which desktop environment or shell they want; never assume a default. If the target OS ships its own shell (a UBports port carries its own, for example), strongly recommend sticking with it — a port that fights the distro's compositor and session stack pays for the fight on every rebuild.
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

Before trusting anything the device reports, check its health:

- **Read the storage's own wear/lifetime report during phase 0 and record it** (UFS life-time percentage, eMMC health register — whatever this storage exposes). Storage is the classic misdiagnosis: on one port an agent read I/O errors as a failing drive, the wear percentage proved the device healthy, and the real fault cleared with one more fastboot flash of the stock ROM. The percentage is what separates "the hardware is dying" from "the software state is bad" — without it you cannot tell those apart, and they have opposite next steps.

Then inventory and harvest the stock system while it is present:

- **Inventory every component the stock system will name** — panel, touch controller, sensor set, camera sensors, modem and its RF configuration, WLAN/BT chip, charger and fuel gauge, audio path — and compare the list against the official spec sheets for this variant (via a web-research or documentation-retrieval skill). Sibling variants differ exactly here, so the comparison is what catches "researched the wrong variant" before it costs a flash.
- **Rooted stock beats a ROM image as a source.** A backup preserves bytes; root keeps the stock system running and readable, so more can be harvested live: the property dump (`getprop`), the stock kernel config (`/proc/config.gz`), the mounted vendor/odm trees, HAL and sensor configs, calibration artefacts, and the factory field-test modes. Assume the user roots the device themselves; like bootloader unlocking, do not provide rooting steps — one wrong step here bricks — and resume once root is available.
- **Record the newest community custom ROM as an optional, up-to-date development source.** Ask whether recent custom ROMs exist for the device and note where they live. Measured: the most responsive OS ever run on one device was an unofficial recent-Android custom build, not the stock ROM — and its boot image shares the stock downstream lineage, so its DTB doubles as an independent cross-check against the phase 0 extraction.

Finally, set the working conventions the rest of the session assumes: a host-side directory that accumulates one subdirectory per boot for logs and pstore, and a note of which recovery path is known to work. If a backup already exists from an earlier attempt, confirm it covers these partitions and that its hashes still verify, rather than assuming it does.

## 1. Gather evidence from the live device first

Capture the failing run's full dmesg before changing anything. Enable the relevant debug knobs _before_ reproducing (e.g. `echo 0xffffffff > /sys/module/<driver>/parameters/debug_mask`), because a second reproduction costs another boot.

Evidence channels, in rough order of reliability:

- **Live shell over the network or USB.** Cheapest channel; use it for anything the device survives.
- **pstore.** Once systemd is up, harvest from `/var/lib/systemd/pstore/`, **not** `/sys/fs/pstore/` — `systemd-pstore.service` _moves_ every record out early in boot, so an empty `/sys/fs/pstore` is not evidence that no crash was recorded. The console region is typically a single slot: copy it to the host before doing anything else, or the next crash overwrites it in the archive too.
- **ramoops.** Treat as lossy. DRAM charge retention decays across the unpowered interval — measured at roughly 6.5 % of bits on one device, enough to fail ECC on the zone headers and lose a record entirely. Grep records _fuzzily_. What rots the region is time spent unpowered, not time spent spamming before you cut power, so let a wedged device log for as long as the console zone can hold, then power-cycle and power back on promptly. Check the zone size before assuming you must hurry: a 4 MiB console zone absorbs hours of watchdog spam before it wraps.
- **Kernel-level USB ACM console** (`CONFIG_U_SERIAL_CONSOLE=y` plus `console=ttyGS0` on the cmdline). The only channel that survives a wedge, and worth its one-time flash cost early in a port. A _userspace_ ACM getty does not survive one — it needs a schedulable CPU, and yields zero bytes while still enumerating fine over USB. Both halves are required: the config symbol alone leaves the facility inert.

Five rules that repeatedly turn inference into measurement:

- **Confirm what is actually flashed by reading the partition back**, never from notes and never from a build log. Read with `iflag=direct` so the read itself does not perturb whatever you are debugging. Beware size-only comparisons — boot images are padded to a fixed size, so equal length across two different images proves nothing. Hash the payload.
- **Reading a debug interface can manufacture the symptom.** DRM/GPU crash-state nodes are the classic case: sampling one while the GPU is actively submitting synthesises fault and recovery lines that never occurred (measured: five reads, five fault/recover pairs, versus none unobserved). Before trusting a log line, check whether your own observation produced it.
- **Capture a rejection's raw response before interpreting the mapped errno.** A firmware call's mapped error was ambiguous — `EINVAL` could mean wrong encoding or refused policy — until dynamic debug in the call wrapper exposed the raw response words: a well-formed call carrying an explicit refusal. That is policy, not encoding — the opposite verdict, with the opposite next step. Classify every firmware rejection as wrong encoding, refused by policy, or not found before acting on it. The mirror rule: a clean, well-formed negative is positive proof — a not-found from a correctly-formed lookup validates the encoding and eliminates a branch of the plan without ever touching the loader.
- **A measurement racing an uncontrolled condition returns a plausible, void number.** One benchmark scored 55 in two seconds: the compositor held DRM master, every scene failed, and the number looked real until the condition was checked. The valid comparison — 132 → 485 — needed the compositor stopped for both runs, compared only within that generation. Before trusting a benchmark, name the condition that could void it and verify it held for every run.
- **Record the rules you withdraw.** Bring-up accumulates folklore fast, and a rule that was right for one hardware revision or one buffer size becomes actively misleading after it changes. Keep the overturned version next to the current one with what changed, so the next session knows which advice has been stress-tested.

Host-side gotchas worth pre-empting:

- `timeout N ssh <device> '<cmd>'` kills the local client only; the remote process keeps running. Follow up with an explicit kill.
- `pgrep -f` and `pkill -f` run over that same ssh path **self-match** their own remote shell. Filter the matcher out, or match by PID.
- `find` needs `-L` when the start path is a symlink, or it returns silently empty.
- Any wrapper or hook that rewrites command output — token filters, pagers, formatters — will corrupt evidence, sometimes by inverting a test rather than obviously breaking it. Use the raw command for anything you will commit as a finding.

Operator hands are a step, not an assumption:

- **When a step needs the operator's physical action** — moving a card between host and device, replugging a cable, holding buttons, confirming something on the device screen — name the exact action and its direction ("move the SD card from the host to the phone"), then **verify it happened before continuing**: an observable signal (device-side mount appears, USB re-enumerates, slot state changes), not the fact that you asked. A port session loses minutes to hours when the agent proceeds on an instruction the operator never registered as an action item.

## 2. Research before implementing — all of these sources

Research always precedes implementation. What it consumes depends on what you are doing:

- **Investigating a failure** — the phase 1 evidence. Capture it first; researching a symptom you have not actually read is guessing with citations.
- **Adding a capability** — a driver, a peripheral, a subsystem that has never come up — the phase 0 artefacts are the device data. The stock DTB, the vendor configs and the firmware layout describe how this board expects the thing to be driven, and they are as authoritative here as a crash log is there.

Either way, **before** writing anything:

1. **Reason about the subsystem** — identify which driver, binding, or firmware interface owns the failure, or would own the new capability. Then read a kernel-level skill by name: invoke `linux-kernel-development` if installed, else `linux-kernel-crash-debug`, else any subsystem-specific kernel skill you have — its register-level and binding detail is exactly what this step needs. If none of them resolves, reason it through directly; do not stall on the missing skill.
2. **Look up userspace libs and tools involved** — current docs, not recalled API. Invoke the `find-docs` skill for each library or tool; if it is not installed, fall back to reading the project's current documentation over the web.
3. **Web research across all seven source families** (parallel subagents work well here). Invoke the `wigolo` skill for the sweep — its cache matters, since sessions re-read the same pages — and if it is not installed, run the sweep with plain web search instead.
   - **The SoC vendor's mainline collaboration project** — for Qualcomm, [github.com/qualcomm-linux](https://github.com/qualcomm-linux). This is the vendor publishing mainline-first kernel work itself: updated dtsi, bindings, subsystem enablement and tooling, often ahead of the nearest mainline release tree. Check it before treating a missing node or driver as work you must author — the vendor may have landed it since, and its trees show how the vendor intends the hardware to be driven under mainline, which is the most authoritative template there is.
   - **Downstream vendor kernel — the GPL-published OEM source.** GPLv2 obliges Android OEMs to publish the kernel sources they shipped, so an exact release for this device usually exists: the OEM's open-source portal (Samsung, Sony, Xiaomi, Fairphone, OnePlus/OPPO and Motorola all run one), their GitHub/GitLab mirrors, and community archives such as XDA when a portal link has died. Select the tarball whose version string matches `/proc/version` from phase 0 — vendors publish several, and they differ. Compliance quality varies, so treat the tree as an oracle to mine — the board dts and defconfig for this exact device, and the out-of-tree vendor drivers (touch, panel, charger, modem glue) with their register sequences and firmware handshakes — rather than assuming it builds. When no exact release surfaces, a sibling device's release usually still carries the SoC dtsi. Better still: **the stock DTB pulled off the device itself** (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`). That is ground truth for _this_ board and has corrected wrong sibling-SoC guesses more than once — including a SMMU stream ID that the sibling SoC got wrong and the stock DTB got right.
   - **postmarketOS** — pmaports device packages and APKBUILDs, merge requests, wiki device pages, and the SoC-mainline tree.
   - **Halium / UBports** — `halium/android_device_*` trees, `hybris-boot` configs, and UBports device ports. Halium runs the Android vendor stack rather than mainline, which makes it the best source for how the _vendor firmware_ expects to be driven: firmware blob paths and load order, the property and socket interfaces daemons expect, sensor and modem HAL configs. When a mainline driver probes but the firmware handshake fails — or when you need to know what handshake a not-yet-supported peripheral expects — this is usually where the missing piece is written down.
   - **Mobian** — the Debian device repos.
   - **NixOS** — mobile-nixos patterns and nixpkgs packaging.

**Thin results are the normal case, not a blocker.** Most of what a port needs has never been done on this board, and often not on this SoC at all. Seven sources reporting nothing means nobody has published the work — it never means the work should stop or be handed back. When precedent is absent, the answers are still in the primary artefacts: the out-of-tree vendor driver that already drives the peripheral (register sequences, firmware handshake, GPIO and regulator wiring), the stock DTB's own node for it, the vendor HAL config naming what userspace expects, and the closest mainline driver for the same device class as a structural template. Read those, derive the design, and write the missing piece — a DT node, a quirk, a small driver. Authoring that piece is the ordinary terminal state of a bring-up, not an exception to be justified; the phase 0 backup and the one-variable-per-flash rule exist precisely so that a wrong derivation costs one boot and one ledger line, the same as any other refutation.

**An empty sweep and an empty scan are different findings.** An empty sweep of the sources means nobody has published the work. An empty _scan_ — an extraction that returns nothing — is void until the same method has been run against a known positive and its coverage is verified: file counts, segment lists, what the glob actually matched. Measured near-misses: a glob matched 9 of 27 firmware segments and nearly recorded a false "the firmware omits it"; a library scan only became trustworthy once it found a known chip id as its positive control.

**Downstream identifiers are downstream's private numbering.** An id or enum in the vendor tree is that tree's own bookkeeping — never copy it into a mainline node. One downstream thermal trip pointed at "sensor 5"; on mainline, sensor 5 read a constant 0, and the real tracker turned out to be sensor 6 — found by stimulus, heating only the suspect cluster and watching which zone moved. When a copied mapping reads dead, suspect the numbering first and settle it by stimulus rather than by more copying.

## 3. Implement only with a promising, evidence-backed hypothesis

- **One variable per flash, falsifier before build.** State the hypothesis before flashing — and go further: write the falsifier before writing any code. The falsifier is the smallest experiment whose result kills the plan either way, and it is built first; one of them — a single module load — returned the refusal that killed a 9.4k-line port premise. Order experiments by cost per information, not by plan order: the cheapest decisive experiment runs first.
- Prefer runtime tests over reflashes: push files with a tar pipe, unbind and rebind drivers, `insmod` a module. Seconds instead of minutes. But a probe leaves state: a failed init can leave its platform device registered, so every retry costs a reboot, and a crashed probe can wedge the board into a physical power-cycle. Stage the probe's inputs and verify their hashes host-side before firing, make the probe idempotent, and expect one reboot per failed attempt.
- Device-tree-only changes are cheap — build just the DTB when the toolchain allows it, then check the runtime surface, not just the build: a DT that builds clean can still bind to nothing (one thermal driver registers one zone per sensor id and silently drops a duplicate, so the zone list, not the build log, is the verdict).
- **Never edit-and-hope.** A change is made by reading the exact current lines and writing the exact replacement — not by pattern-guessing and then "repairing" the syntax errors the first attempt produced. An agent that announces it is fixing its own broken edit has already cost a loop iteration; on a device port that sloppy edit can also reach a flash before it reaches a build. If a tool cannot make the edit precisely, stop and re-read; hand-repair of self-inflicted breakage is a defect, not a workflow.
- **A rebuild is the most expensive step in the loop — verify the change before paying for it.** A full kernel rebuild costs tens of minutes, and a build that dies 15 minutes in on a missing symbol taught you what a 10-second host-side check would have. Measured: four consecutive rebuilds were burned on missing symbols in patches that no one checked before building. Before any full rebuild, run the cheap battery host-side: every patch applies cleanly to the exact tree (`git apply --check`); every symbol a patch references is defined somewhere the patch can see (grep the tree — a symbol renamed upstream or only present in another directory kills the build late); every config option a patch or defconfig sets actually exists in this kernel's Kconfig — an option that does not exist is silently ignored, so a clean config read is not evidence it took effect; and compile the touched directories or translation units first, so the expensive full build only ever runs on code that already compiled alone.
- **After 3 refuted fixes: stop guessing, not stop working.** Three misses from the same model of the failure mean the model is wrong, and a fourth tweak from it will miss too. Return to step 2 with a wider scope — a different layer of the stack, or the primary artefacts directly where precedent is absent — and come back with findings stacked, as in the worked example below.

A worked shape, from a Wi-Fi bring-up that took this path: five blind device-tree tweaks all failed. The fix was five separate findings stacked, every one of them from research rather than guessing — a trustzone memory mode, vendor firmware paths, a host-capability quirk, the SMMU stream ID off the stock DTB, and one property that mainline had made obsolete and needed dropping. No single tweak would have got there.
