---
description: Research a symptom or a new capability across all six source families before writing any code (phase 2 of linux-phone-porting)
argument-hint: "[symptom, or the capability you are about to build]"
---

Run phase 2 of the `linux-phone-porting` skill for: $ARGUMENTS

**Precondition — whichever applies:**

- **Investigating a failure:** its evidence must already be captured. If it is not, stop and run `/evidence-sweep` first; this command is not a substitute for reading the device.
- **Adding a capability** that has never worked on this board: the phase 0 artefacts are the input — stock DTB, vendor configs, firmware blob layout. If they were never extracted, run `/port-setup` first.

Your session context lists the skills available to you. Scan that list first, then load each of these that appears in it — skip silently only when absent:

- a kernel-development or crash-analysis skill, to identify the owning driver, binding, or firmware interface
- a documentation-retrieval skill, for any userspace library or tool involved
- a web-research skill, for the source sweep below — prefer one with a local cache, since these pages get re-read across sessions

Then sweep **all six** source families, in parallel subagents where possible. The point is that the fix comes from all of them, not from the first plausible hit:

1. **Mainline** — lore.kernel.org, the driver source, DT bindings, and the dtsi of the closest sibling SoC. Before scoping a port of a downstream driver, establish what mainline already has for the interface: its exported API, whether it reaches userspace or is kernel-internal only, and its gates (machine allowlists, pre-loaded-by-bootloader semantics, built-in rather than module) — one such sweep has cut a 9.4k-line port down to a ~300-line probe.
2. **Downstream vendor kernel — the GPL-published OEM source.** GPLv2 obliges Android OEMs to publish the kernel sources they shipped, so an exact release for this device usually exists — OEM open-source portals, their GitHub/GitLab mirrors, XDA/community archives when a portal link has died. Match the tarball to the `/proc/version` fingerprint from phase 0, and mine it as an oracle (board dts, defconfig, out-of-tree vendor drivers) rather than assuming it builds; a sibling device's release is the fallback when no exact one surfaces. Above all the **stock DTB read off this device** (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`). Where the sibling SoC and the stock DTB disagree, the stock DTB wins. Treat the tree's identifiers as its private numbering: an id or enum in the vendor DT is never copied into a mainline node — verify a mapping by stimulus when it reads dead on mainline.
3. **postmarketOS** — pmaports device packages and APKBUILDs, merge requests, wiki device pages, the SoC-mainline tree.
4. **Halium / UBports** — `halium/android_device_*`, `hybris-boot` configs, UBports device ports. Best source for the vendor firmware handshake: blob paths and load order, expected properties and sockets, HAL configs.
5. **Mobian** — the Debian device repos.
6. **NixOS** — mobile-nixos patterns and nixpkgs packaging.

Report per source: what it says about this symptom, or explicitly that it had nothing. An empty source is a finding, not a failure — but an empty _scan_ is a different thing: an extraction that returns nothing is void until the same method has found a known positive and its coverage is verified (file counts, segment lists, what the glob actually matched).

**Before designing any driver, establish who owns the hardware — from the phase 0 images, not from assumptions.** Scan the backed-up partitions for the peripheral's driver symbols — strings/ELF scans of the DSP, TrustZone and modem firmware images and the vendor HAL objects — and diff that against the kernel-side artefacts. This has flipped plans at the design step: one kernel side that looked like the driver was a GPIO/IRQ/netlink shim with the real driver a TrustZone applet inside a firmware image, which turned "port a driver" into "load the applet"; another peripheral's algorithm lived in DSP firmware, unreachable from Linux, which closed the feature outright.

When all six come back empty, derive the candidate hypotheses from the primary artefacts — the vendor driver source, the stock DTB, the closest mainline sibling driver — and rank them like any other.

Finish with the hypotheses ranked by evidential support, and name which source backs each. For each hypothesis also state its falsifier: the smallest experiment whose result kills it either way — so the implement phase starts with the cheapest decisive probe, not the plan's first step.
