---
description: Research a symptom or a new capability across every source family before writing any code (phase 2 of linux-phone-porting)
argument-hint: "[symptom, or the capability you are about to build]"
---

Run phase 2 of the `linux-phone-porting` skill for: $ARGUMENTS

**First: refresh the session's task list** — replace whatever is there with this command's steps, one item per step, first step in progress (`SKILL.md` chain rule).

**Precondition — whichever applies:**

- **Investigating a failure:** its evidence must already be captured. If it is not, stop and run `/evidence-sweep` first; this command is not a substitute for reading the device.
- **Adding a capability** that has never worked on this board: the phase 0 artefacts are the input — stock DTB, vendor configs, firmware blob layout. If they were never extracted, run `/port-setup` first.

Load these skills by name, in this order — invoke each; if it is not installed, skip it and use the fallback (the skill works without them):

- `linux-kernel-development` (else `linux-kernel-crash-debug`), to identify the owning driver, binding, or firmware interface — fallback: reason through the driver sources directly.
- `find-docs`, for any library, tool, or kernel interface involved — kernel-side too: the identified driver's DT binding, Kconfig options and subsystem documentation for the exact kernel version being built — fallback: read the project's current documentation over the web, or `Documentation/` in the kernel tree for kernel interfaces.
- `wigolo`, for the source sweep — its cache matters, since these pages get re-read across sessions; fallback: plain web search.

Then run **phase 2 in full**: the selected target platform's requirements and discovered related projects, ordered source families, component-document mapping, wiring reconciliation, identity/codename and archive searches, version-matched advisory/source checks, and optional `graft` navigation are authoritative in `SKILL.md`, not duplicated here. Named projects are research destinations, not a target compatibility list. Use parallel subagents where possible. The point is that the fix comes from all sources, not the first plausible hit. Apply the skill's scoped stock-DT authority: establish the selected variant/overlay, distinguish configuration from physical identity or measured wiring, and preserve unresolved conflicts rather than declaring one source universally decisive.

**Sweep mechanics — self-executing, ledger-gated:**

- **The sweep starts now.** Precondition met means no further operator input: do not wait to be told to search, which tool to use, or where to look. `SKILL.md`'s phase 2 is the procedure; this command is only its entry point.
- **Open a sweep ledger** in the project's research notes: one row per open question × source family, each row carrying the consulted revision/query, the finding + applicability, or a recorded gap (source + access failure). The next session inherits coverage, not a re-search.
- **Fan out by default** — one parallel subagent per source family, each returning per-question findings **or** explicit gaps. A subagent that comes back silent has recorded a gap to re-ask, not a negative result.
- **Exit criterion:** the sweep is done only when every applicable family row carries a finding or a recorded gap. First plausible hit ≠ done. Only then reconcile records, rank hypotheses, attach falsifiers.

**Then sweep sister devices within that procedure.** Enumerate ports of the recorded device family and near-neighbours on the same board or SoC generation, including internal codenames rather than only retail names. Diff each port's board files (dts, defconfig, firmware blob lists) component-by-component against the phase 0 inventory: matching components yield proven driver-plus-firmware combinations; non-matching ones mark divergence. Adopt only what the diff confirms against this variant's spec sheet and stock DTB. A sister device is a lead, not proof of target compatibility; a same-SoC rescue image or loader remains an uncertified candidate.

Report per source: the relevant finding and applicability, or no applicable result within the inspected revision/coverage. Record inaccessible sources as gaps, not negative evidence. An empty _scan_ is different: an extraction that returns nothing is void until the same method has found a known positive and its coverage is verified (file counts, segment lists, what the glob actually matched). Read success claims in context before treating them as precedent for this capability.

Update the affected component and document records using phase 2's contract; link each finding to its source location, acquisition state and applicability. Report unresolved wiring/identity conflicts and document gaps before ranking hypotheses.

**Before designing any driver, establish who owns the hardware — from the phase 0 images, not from assumptions.** Scan the backed-up partitions for the peripheral's driver symbols — strings/ELF scans of the DSP, TrustZone and modem firmware images and the vendor HAL objects — and diff that against the kernel-side artefacts. This has flipped plans at the design step: one kernel side that looked like the driver was a GPIO/IRQ/netlink shim with the real driver a TrustZone applet inside a firmware image, which turned "port a driver" into "load the applet"; another peripheral's algorithm lived in DSP firmware, unreachable from Linux, which closed the feature outright. Silence from a DSP can be an unanswered callback rather than dead silicon — one DSP called back into the application processor to read its own configuration files and silently did nothing until a file server answered, so before concluding an applet is absent, check what AP-side files it expects and whether anything serves them. And a missing fused capability is not a missing sensor: every physical sensor can work while every fused algorithm is absent — strings-scan the harvest for the exact identifier your falsifier saw refused, which is how one refused TrustZone applet was matched to the stock userspace that owns it.

When no applicable precedent is found, derive candidate hypotheses from the primary artefacts — the vendor driver source, the stock DTB, the closest mainline sibling driver — and rank them like any other, retaining search limitations.

If the finding appears to be a shared upstream defect, record the exact affected and candidate-fixed revisions, captured reproduction and boundary conditions for the phase 3 report. Qualify advisory/CVE applicability by matching source and preconditions, not merely a similar symptom; distinguish submitted, accepted, merged and released fixes.

Finish with the hypotheses ranked by evidential support, and name which source backs each. For each hypothesis also state its falsifier: the smallest experiment whose result kills it either way — so the implement phase starts with the cheapest decisive probe, not the plan's first step.

**Then** phase 3 starts on the sweep's output, not on a separate request: carry the top hypothesis and its falsifier into `/flash-gate` yourself. If nothing is implementable — every hypothesis blocked on a missing artefact, permission or capability — stop there and report the ranked open questions plus the one experiment or acquisition that would unblock the first of them.
