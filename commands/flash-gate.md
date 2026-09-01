---
description: Gate a fix before it costs a flash (phase 3 of linux-phone-porting)
argument-hint: "[the fix you are about to apply]"
---

Apply phase 3 of the `linux-phone-porting` skill to: $ARGUMENTS

Answer each of these before anything is written to the device. Any "no" blocks the flash.

1. **What is the hypothesis?** State it as a falsifiable sentence — what is broken, why this change addresses it, and what the device will do differently if it is right.
2. **Which source backs it?** Name the evidence or the research finding. "It seems likely" is a refusal. A design derived from the vendor driver source or the stock DTB passes — naming the artefact is naming the source. That no distro has done this before is not a refusal; it is the normal case in bring-up.
3. **Is this one variable?** If the change touches more than one thing, split it. A passing multi-variable flash teaches you nothing about which part worked.
4. **Can it be tested without a flash?** Push files with a tar pipe, unbind and rebind the driver, `insmod` the module, rebuild only the DTB. Seconds beat minutes; take the cheap test if one exists.
5. **How many fixes have been refuted on this symptom?** At three, stop. Do not flash a fourth — go back to `/source-sweep` with a wider scope, because the model of the failure is what is wrong.
6. **What result would refute this?** Decide now, so the next boot's dmesg is read honestly rather than for confirmation.

If all six pass, flash, then record the outcome against the hypothesis as stated — including, on a refutation, what it rules out.
