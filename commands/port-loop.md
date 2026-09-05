---
description: Iterate research → implement → verify autonomously until a capability works or the model of it is exhausted (linux-phone-porting)
argument-hint: "[the feature or capability you want working]"
---

Get this working on the device, iterating until it does: $ARGUMENTS

This runs the `linux-phone-porting` phases in a loop rather than one at a time. It does not replace them — each iteration calls them, and their rules still bind. Do not restate the seven sources here; `/source-sweep` owns them. Do not edit-and-hope either: every change lands by reading the exact current lines and writing the exact replacement — an edit that has to be "repaired" afterwards is redone from a fresh read, not patched over.

**Before the first iteration.** Confirm the phase 0 artefacts exist — stock DTB, vendor configs, firmware layout, verified backup with hashes. If they do not, run `/port-setup` and stop; there is nothing to research against, and the loop would research some other board that shares this SoC. State the target as one falsifiable sentence: what the device will do that it does not do today, and how you will observe it.

**Each iteration.**

1. **Research.** `/port-research` on the current framing of the gap. On the first iteration that is the whole capability; afterwards it is whatever the last refutation left unexplained.
2. **Gate.** `/flash-gate` the smallest change that would produce a legible result. All six questions pass or the change is not ready — in particular, one variable, and a stated result that would refute it.
3. **Apply.** Prefer the cheap path every time: tar-pipe files, unbind/rebind, `insmod`, rebuild only the DTB. Flash only when nothing cheaper can test the hypothesis.
4. **Verify.** `/evidence-sweep` on the result. Read it against the refutation criterion written in step 2, not for confirmation. If a step needed the operator's hands (moving a card, replugging, a button hold), name the exact action and direction when asking, and confirm it from an observable signal before continuing — never proceed on the assumption the request was carried out.
5. **Record.** One line per iteration: hypothesis, source that backed it, what was applied, outcome, and on a refutation what it rules out. This ledger is the loop's memory — a later iteration that re-proposes a refuted change is a bug.

**Stop and hand back to the human when any of these is true.**

- The target sentence is satisfied, verified from device evidence rather than from the absence of an error.
- **Three refutations on the same symptom.** Widen the scope once — re-enter `/port-research` on the layer above, or on the primary artefacts directly where no precedent exists, not the same layer again — and if that round also refutes, stop. The model of the failure is wrong and more iterations of it will not find it.
- The next step would write to the bootloader, modem NV/EFS, persist, the partition table, or anything else whose loss is not recoverable from the phase 0 backup. Boot, dtbo, modules and the like are yours to flash; these are not. Say what you want to write and why, and wait.
- The device stops responding on every evidence channel, or a flash leaves it unable to reach a known-good recovery path. Do not attempt further writes blind.
- Neither precedent nor derivation yields a testable next step: every source returns what the ledger already contains, and the primary artefacts — the vendor driver source, the stock DTB, the closest mainline sibling driver — offer nothing that can be stated as a falsifiable change with a named source behind it. An empty sweep alone is not this condition: empty precedent hands the work to the primary artefacts, and the loop continues.

Report at the end: the ledger, the current state of the device, and either what made it work or what the refutations collectively rule out.
