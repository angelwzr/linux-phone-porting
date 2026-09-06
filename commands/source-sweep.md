---
description: Research a symptom or a new capability across every source family before writing any code (phase 2 of linux-phone-porting)
argument-hint: "[symptom, or the capability you are about to build]"
---

Run phase 2 of the `linux-phone-porting` skill for: $ARGUMENTS

**Precondition — whichever applies:**

- **Investigating a failure:** its evidence must already be captured. If it is not, stop and run `/evidence-sweep` first; this command is not a substitute for reading the device.
- **Adding a capability** that has never worked on this board: the phase 0 artefacts are the input — stock DTB, vendor configs, firmware blob layout. If they were never extracted, run `/port-setup` first.

Load these skills by name, in this order — invoke each; if it is not installed, skip it and use the fallback (the skill works without them):

- `linux-kernel-development` (else `linux-kernel-crash-debug`), to identify the owning driver, binding, or firmware interface — fallback: reason through the driver sources directly.
- `find-docs`, for any userspace library or tool involved — fallback: read the project's current documentation over the web.
- `wigolo`, for the source sweep — its cache matters, since these pages get re-read across sessions; fallback: plain web search.

Then sweep **every source family** listed in phase 2 of the skill, in parallel subagents where possible. The point is that the fix comes from all of them, not from the first plausible hit. One authority rule repeated here because it decides the sweep's outcome: the stock DTB read off this device wins wherever any source — including the sibling SoC's dtsi and the OEM tree's board dts — disagrees.

**Then sweep sister devices.** Within the sources, enumerate ports of this device's family — sibling models sold under the same marketing name (the codenames phase 0 recorded), and near-neighbours on the same board or SoC generation. pmaports device pages, Halium/UBports device trees, XDA device forums, and the OEM kernel tree's own `dts` directory are where they surface. Diff each port's board files (dts, defconfig, firmware blob lists) component-by-component against the phase 0 inventory: matching components yield proven driver-plus-firmware combinations; non-matching ones mark where the family diverges. Adopt only what the diff confirms against this variant's spec sheet and stock DTB — a sister device is a lead, not a precedent, because sibling variants differ exactly in panel, touch, and modem.

Report per source: what it says about this symptom, or explicitly that it had nothing. An empty source is a finding, not a failure — but an empty _scan_ is a different thing: an extraction that returns nothing is void until the same method has found a known positive and its coverage is verified (file counts, segment lists, what the glob actually matched).

**Before designing any driver, establish who owns the hardware — from the phase 0 images, not from assumptions.** Scan the backed-up partitions for the peripheral's driver symbols — strings/ELF scans of the DSP, TrustZone and modem firmware images and the vendor HAL objects — and diff that against the kernel-side artefacts. This has flipped plans at the design step: one kernel side that looked like the driver was a GPIO/IRQ/netlink shim with the real driver a TrustZone applet inside a firmware image, which turned "port a driver" into "load the applet"; another peripheral's algorithm lived in DSP firmware, unreachable from Linux, which closed the feature outright. Silence from a DSP can be an unanswered callback rather than dead silicon — one DSP called back into the application processor to read its own configuration files and silently did nothing until a file server answered, so before concluding an applet is absent, check what AP-side files it expects and whether anything serves them. And a missing fused capability is not a missing sensor: every physical sensor can work while every fused algorithm is absent — strings-scan the harvest for the exact identifier your falsifier saw refused, which is how one refused TrustZone applet was matched to the stock userspace that owns it.

When every source comes back empty, derive the candidate hypotheses from the primary artefacts — the vendor driver source, the stock DTB, the closest mainline sibling driver — and rank them like any other.

Finish with the hypotheses ranked by evidential support, and name which source backs each. For each hypothesis also state its falsifier: the smallest experiment whose result kills it either way — so the implement phase starts with the cheapest decisive probe, not the plan's first step.
