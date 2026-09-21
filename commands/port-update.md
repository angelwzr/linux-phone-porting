---
description: Bring the port's kernel, firmware, userspace and other components to newer releases in one governed pass (linux-phone-porting)
argument-hint: "[component targets, or 'latest' for everything pinned]"
---

Update the port's components in one pass: $ARGUMENTS

This is a batch orchestrator over rules that live elsewhere; it adds sequencing, rollback anchors and a ledger, not new rules. `SKILL.md` phase 3 (separate restructuring from upgrades, the pre-build battery, the deployment state machine, the cleanup gate), `/flash-gate`, `/evidence-sweep` and `/port-cleanup` bind every step below.

**Precondition:** phase 0 complete; the currently deployed system known-good and bootable, its identity verifiable through the demonstrated readback; and a complete rollback path for every component about to change — an update that cannot be rolled back is not an update, it is a redeploy. Workspace clean; no uncommitted device state.

**1. Inventory the current state — the rollback anchor.** For every updatable component record: name, pinned version/revision (flake lock, APKBUILD tag, package version, firmware blob hash), the artifact it produces, and the command that verifies it. Read back the deployed identities now and write the anchor into an update ledger file — prose holds no numbers. Confirm the rollback path actually restores; a saved image alone may not recover the system.

**2. Frame the targets, ask once.** Enumerate the candidates: kernel lineage, SoC/common patch-series rebase, device DT and patches, firmware packages, userspace packages and apps, OS platform/channel, and bootloader-adjacent artefacts where the class has them. For each: current → target, and the reason. Before proposing targets, sweep: known regressions in the target kernel version (LKML/lore), the target project's release and upgrade notes, sibling ports already on the target version. Present the list with dependency notes and get the operator's confirmation of scope and order — the target list is an operator decision, like the loop budget.

**3. Order by dependency; one component class per deployment cycle.** Never combine a kernel-version bump with a layout or restructure move, and never bundle unrelated upgrades into one write — a passing multi-component update teaches nothing about which part worked. Typical order: kernel lineage (series rebase + config migration) → SoC/common series → device DT/patches → firmware → userspace/apps → platform channel. Each class: build → gate → deploy → verify before the next begins.

**4. Per component:**

- **Patches:** fresh disposable tree at the target revision, patches applied in series order; check each for upstream absorption — a patch the new version already contains is retired with evidence (upstream commit link), not silently carried; conflicts are re-derived from the new base, never forced.
- **Config:** run the resolved-config check; options renamed or removed by the new version are migrated explicitly — a silently ignored option is a change you did not make.
- **Build:** the pre-build battery before any full rebuild; select artifacts explicitly, report per artifact.
- **Write/deploy:** `/flash-gate` for every device write; the deployment state machine observed separately (transfer, next-boot selection, activation, observed reboot).
- **Verify:** the component's own function AND a regression pass over capabilities that already worked — one upgrade lost sound-card registration to module coldplug ordering while later manual init worked; earlier wins are part of the acceptance surface. `/evidence-sweep` captures before/after under the same controlled conditions.
- **Ledger:** one line per component: from → to, patch retirements with evidence, verification results, the rollback anchor for this step.

**5. Stop and hold when:** a verification fails after the standard bounded retries (hold = roll that component back to its anchor or pause the batch — state which in the ledger); the same refutation repeats three times on one component → `/port-research` on it, not a fourth attempt; the operator defers permission; the next step's rollback path cannot be proven.

**6. Close.** Reconcile the ledger with the completion-record rule — current state, tested revisions, acceptance results, limitations, authorization. Update the project's authority pointers the update moved (flake lock, APKBUILD, pinned revisions). Then `/port-cleanup` retires the orphaned kernels, generations, images and store closures — only after the batch passes real reboot + runtime checks.

Report: the ledger, per-component outcomes, what was retired and why (absorption evidence), rollback anchors retained, and any component deliberately left behind with the reason.
