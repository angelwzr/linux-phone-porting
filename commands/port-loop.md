---
description: Iterate research → implement → verify autonomously until a capability works or the model of it is exhausted (linux-phone-porting)
argument-hint: "[the feature or capability you want working]"
---

Get this working on the device, iterating until it does: $ARGUMENTS

This runs the `linux-phone-porting` phases in a loop rather than one at a time. It does not replace them — each iteration calls them, and their rules still bind. `SKILL.md` owns the phase rules and source families; the phase commands apply them. Skill maintenance is outside this hardware loop: do not auto-update or change the ruleset mid-loop.

**Before the first iteration.** Confirm phase 0 completed under `/port-setup`, with its target, artifact, backup and control-channel prerequisites available. If not, run `/port-setup` and stop; there is nothing to research against, and the loop would research some other board that shares this SoC. State the target as one falsifiable sentence: what the device will do that it does not do today, and how you will observe it. Then ask the operator, **at every invocation — the budget is a choice for this session, never carried over from a previous one nor recorded as a project setting**: how many refutations is this loop budgeted — **5, 15, or unlimited**. Record the answer only in this session's ledger header; every later stop decision reads against it.

**Each iteration.**

1. **Research.** `/port-research` on the current framing of the gap. On the first iteration that is the whole capability; afterwards it is whatever the last refutation left unexplained. The refutation narrowing the question does **not** waive the sweep: re-run it against the narrowed framing (the wigolo cache makes the re-read cheap), and the pass must return at least one finding the ledger does not already contain — a source family not yet read on this question, a newly read part of the vendor driver or stock DTB, or a documented no from a source that previously said nothing. If it cannot, the iteration's hypothesis may still proceed, but `/flash-gate` question 2 will demand a source from this pass — plan for that, not for the ledger.
2. **Gate.** `/flash-gate` the smallest change that would produce a legible result. Its prerequisites and all six questions must pass; record its falsifier before applying anything.
3. **Apply.** Execute the approved experiment through phase 3, including its target-write checks, cheapest-path selection, reboot/operator-hand verification and any post-upgrade cleanup gate. Do not substitute a direct flash for those rules.
4. **Verify.** `/evidence-sweep` on the result. Read it against the refutation criterion written in step 2, not for confirmation; apply phase 3's end-to-end acceptance criteria, not merely reboot or service readiness.
5. **Record.** One line per iteration: hypothesis, source that backed it, what was applied, outcome, and on a refutation what it rules out. This ledger is the loop's memory — a later iteration that re-proposes a refuted change is a bug.

**Stop and hand back to the human when any of these is true.**

- The target sentence is satisfied, verified from device evidence rather than from the absence of an error.
- **The refutation budget is exhausted.** The budget comes from the operator at invocation (5, 15, or unlimited) and lives in the ledger header. Widening is not a stop: every third refutation on the same symptom forces a scope change — re-enter `/port-research` on the layer above, or on the primary artefacts directly where no precedent exists, never the same layer again — and the loop continues while budget remains. On a finite budget, stop when it is spent and hand back what the refutations collectively rule out; on unlimited, keep looping — the other stop conditions below are the only exits.
- The next step would write to the bootloader, modem NV/EFS, persist, the partition table, or anything else whose loss is not recoverable from the phase 0 backup. Boot, dtbo, modules and the like are yours to flash; these are not. Say what you want to write and why, and wait.
- The device stops responding on every evidence channel, or a flash leaves it unable to reach a known-good recovery path. Do not attempt further writes blind.
- Neither precedent nor derivation yields a testable next step: every source returns what the ledger already contains, and the primary artefacts — the vendor driver source, the stock DTB, the closest mainline sibling driver — offer nothing that can be stated as a falsifiable change with a named source behind it. An empty sweep alone is not this condition: empty precedent hands the work to the primary artefacts, and the loop continues.

Report at the end: the ledger, the current state of the device, and either what made it work or what the refutations collectively rule out.
