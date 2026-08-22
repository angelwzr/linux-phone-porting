---
description: Research a requested change against the device and all six sources before building it (phase 2 of linux-phone-porting, capability framing)
argument-hint: "[what you have been asked to build or enable]"
---

Research before implementing: $ARGUMENTS

This is `/source-sweep` entered from the other side. Use it when the request is to **build or enable something**, rather than to explain something that broke — a peripheral that has never worked, a driver to wire up, a subsystem to bring up for the first time. Same procedure, same six sources; what differs is the input and the output.

**Input.** There is no crash log, so the device data is the phase 0 material: the stock DTB, vendor sensor/modem/HAL configs, firmware blob paths and load order, the vendor kernel cmdline. If those were never extracted, run `/port-setup` first — researching this board without them means researching some other board that shares its SoC.

**Procedure.** Run `/source-sweep` in full. Read every question it asks about "the failure" as a question about the capability instead: which driver or binding would own it, what the vendor stack does today, what mainline already supports.

**Output.** Not a hypothesis about a symptom, but a plan with its sources attached:

- What mainline already provides, and what is missing.
- What the vendor stack does instead, from the stock DTB and Halium/UBports configs — the interface any mainline path will have to satisfy.
- Which other distro has done this on this SoC, and what they had to add.
- The smallest first change that would produce a legible result, and what result would tell you the model is wrong.

Do not write the change in this command. It ends at the plan, and `/flash-gate` still applies before anything reaches the device.
