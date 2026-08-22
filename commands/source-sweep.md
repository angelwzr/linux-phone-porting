---
description: Research a symptom or a new capability across all six source families before writing any code (phase 2 of linux-phone-porting)
argument-hint: "[symptom, or the capability you are about to build]"
---

Run phase 2 of the `linux-phone-porting` skill for: $ARGUMENTS

**Precondition — whichever applies:**

- **Investigating a failure:** its evidence must already be captured. If it is not, stop and run `/evidence-sweep` first; this command is not a substitute for reading the device.
- **Adding a capability** that has never worked on this board: the phase 0 artefacts are the input — stock DTB, vendor configs, firmware blob layout. If they were never extracted, run `/port-setup` first.

Check which of these are installed before calling them, and skip silently if absent:

- a kernel-development or crash-analysis skill, to identify the owning driver, binding, or firmware interface
- a documentation-retrieval skill, for any userspace library or tool involved
- a web-research skill, for the source sweep below — prefer one with a local cache, since these pages get re-read across sessions

Then sweep **all six** source families, in parallel subagents where possible. The point is that the fix comes from all of them, not from the first plausible hit:

1. **Mainline** — lore.kernel.org, the driver source, DT bindings, and the dtsi of the closest sibling SoC.
2. **Downstream vendor kernel** — the OEM release for a sibling device, and above all the **stock DTB read off this device** (`dd` the untouched boot slot, scan for `d00dfeed`, `dtc -I dtb -O dts`). Where the sibling SoC and the stock DTB disagree, the stock DTB wins.
3. **postmarketOS** — pmaports device packages and APKBUILDs, merge requests, wiki device pages, the SoC-mainline tree.
4. **Halium / UBports** — `halium/android_device_*`, `hybris-boot` configs, UBports device ports. Best source for the vendor firmware handshake: blob paths and load order, expected properties and sockets, HAL configs.
5. **Mobian** — the Debian device repos.
6. **NixOS** — mobile-nixos patterns and nixpkgs packaging.

Report per source: what it says about this symptom, or explicitly that it had nothing. Finish with the candidate hypotheses ranked by evidential support, and name which source backs each.
