---
description: Research a requested change against the device and all seven sources before building it (phase 2 of linux-phone-porting, capability framing)
argument-hint: "[what you have been asked to build or enable]"
---

Research before implementing: $ARGUMENTS

This is `/source-sweep` entered from the other side. Use it when the request is to **build or enable something**, rather than to explain something that broke — a peripheral that has never worked, a driver to wire up, a subsystem to bring up for the first time. Same procedure, same seven sources; what differs is the input and the output.

**Input.** There is no crash log, so the device data is the phase 0 material: the stock DTB, vendor sensor/modem/HAL configs, firmware blob paths and load order, the vendor kernel cmdline. If those were never extracted, run `/port-setup` first — researching this board without them means researching some other board that shares its SoC.

**Procedure.** Run `/source-sweep` in full. Read every question it asks about "the failure" as a question about the capability instead: which driver or binding would own it, what the vendor stack does today, what mainline already supports.

**Output.** Not a hypothesis about a symptom, but a plan with its sources attached:

- Who owns the hardware for it — a kernel driver, a firmware applet, or DSP-side code — answered by scanning the phase 0 images for the peripheral's symbols rather than assumed from the kernel side. A kernel-side shim with the real driver in firmware turns the plan from porting a driver into loading one.
- What mainline already provides — its exported API, whether it reaches userspace or is kernel-internal only, and its gates (machine allowlists, pre-loaded semantics, built-in rather than module) — and what is missing.
- What the vendor stack does instead, from the stock DTB and Halium/UBports configs — the interface any mainline path will have to satisfy.
- Which other distro has done this on this SoC, and what they had to add.
- If no distro has, say so plainly and derive the plan from the primary artefacts instead — the vendor driver source, the stock DTB, the closest mainline sibling driver. Absence of precedent is the normal state of a bring-up, not a dead end.
- The smallest first change that would produce a legible result, stated with its falsifier — the cheapest experiment whose result kills the plan either way. That probe is built before anything larger.

Do not write the change in this command. It ends at the plan, and `/flash-gate` still applies before anything reaches the device.
