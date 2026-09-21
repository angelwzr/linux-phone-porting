---
description: Use when disk pressure or an explicit operator request calls for cleaning a Linux-phone porting workspace — stale artifacts, old shared kernel/OS bases, abandoned build outputs, stale result links and package-store closures (host-side workspace maintenance, outside the hardware phases)
argument-hint: "[device project, shared base, or workspace root]"
---

Reclaim host-side disk, with proof and measurement: $ARGUMENTS

**First: refresh the session's task list** — replace whatever is there with this command's steps, one item per step, first step in progress (`SKILL.md` chain rule).

Workspace maintenance only — never touches the phone. Device-side cleanup keeps `SKILL.md` phase 3's gate and is out of scope here.

**Preconditions — stop without them.**

- Operator asked explicitly. Never mid-`/port-loop`, mid-experiment, or during a build/deploy.
- Current state is recorded: agent context files name the tested revision; an artifact manifest (`artifacts/README.md` or equivalent) maps artifact paths to classes. Unmanifested artifact = unknown — classify with the operator first; it cannot be proposed this run.
- Last deployment passed acceptance checks.
- Anything ambiguous ⇒ keep the item and say so. Deletion runs on recorded state plus evidence, never inference.

1. **Measure.** `df` per filesystem in scope; `du` per candidate. Every proposed size is measured. Split the accounting: file deletions free immediately; removing a result link frees nothing until the package manager's garbage collection reclaims unreferenced closures — label those estimates.

2. **Inventory by class.**

   - **Build outputs** — Kbuild objects/modules/`.cmd`/`.tmp_versions` in tracked module dirs, boot-image unpack staging, generated docs, local caches. Candidate when tracked inputs regenerate them.
   - **Result links** — classify from deploy records, not names: the link the last deployment consumed (keep), older iterations (stale), dead targets (dangling), build-output links no deployment ever consumed (stale — they pin store closures nothing else references).
   - **Shared bases** — `kernel/<version>/`, `soc/*`, `os/*` not required by any consumer. In use if any consumer flake/lock pins them, or the recorded known-good recovery build needs them. Enumerate pins; never infer from directory names.
   - **Logs** — keep unconditionally: every `LATEST-*` target, the current boot's log, anything docs or ledgers cite. Propose the rest oldest-first; operator sets the retention count.
   - **Artifacts — one decision per manifest record:**
     - reproducible packages: deletable when hashes are recorded and re-acquisition is credible — state how.
     - reference copies: deletable when re-acquirable; "findable again" is a claim to check.
     - harvest blobs: deletable only when no tracked package references the file/hash, and a redacted copy exists or the operator condemns them.
     - PROTECTED — device-unique private material, irreplaceable evidence, git time capsules: never proposed, never batch-approved. Removal is a separate per-item operator decision outside this run.
     - manifest removal conditions: met = evidence; unmet = keep.
   - **Strays** — root-level packages/dumps/images: relocate into the right `artifacts/` class plus a manifest row, not deletion.

3. **Prove non-use — the gate.** Deletable only with recorded evidence the current port does not reference it. Sweep: agent context files, flake inputs and locks, package definitions (file/hash references), tools and scripts, docs, ledgers, the manifest, `LATEST-*` links.

   - Validate the sweep against a reference known to exist before trusting a negative — a search matching nothing anywhere is void.
   - A hit means keep; updating a stale reference is part of the plan, never silent.
   - Store closures: ask the package manager who consumes a path; the recovery closure is always a consumer.
   - Record what was checked per "unused" verdict — that record is the proposal's evidence column.

4. **Propose, confirm per item.** Table: item, class, action (delete / unlink+GC / relocate / keep), measured size, evidence, cost if wrong. Include totals split by mechanism and a kept-and-large list (protected, current deployment, recovery path). Ask; wait. Blanket "delete everything" is refused; silence or a partial answer leaves unconfirmed items in place. Disk changed since step 1 ⇒ re-measure, re-propose.

5. **Execute in order.** Relocations + manifest edits → stale/dangling links, approved logs, build outputs → base trees last, after every consumer pin naming them is re-pointed and re-locked → store content only via the package manager's scoped reclamation of unreachable paths. Never manual store deletion; never blanket GC that can eat the protected recovery closure — pin every closure that must survive reclamation (the recorded recovery path, the deployed system, the known-good rollback) as an explicit package-manager GC root, in a recorded location named for its purpose, and verify every pin still resolves after; on unsure GC flags `find-docs` the tool's docs, else its manual. Mid-run discoveries return to step 4.

6. **Verify.**

   - **References:** re-sweep every removed path — zero live hits; a hit now is a failed evidence step: reinstate from the store or record the loss plainly. Kept `LATEST-*` and result links resolve; every flake input and lock entry resolves to an existing tree.
   - **Builds:** flake eval/check proves inputs, pins and package definitions resolve — state what eval does not prove; build-reachable removals require building the consuming package (or the operator's named target), never a full image unless asked.
   - **State:** the run's tree changes — manifest rows, ignore rules, tracked deletions, relocations — are committed before the report; a cleanup that leaves the working tree dirty has half-executed itself.
   - **Paths:** manifest old→new map updated; no doc, script or lock names a missing path; no dangling result link remains in the swept roots — a dead symlink is a removal that did not happen.

7. **Report freed space.** Persist the report as a file in the project's records, not only in conversation — a session ends, and the next run must not need archaeology to learn what was removed, freed, kept, and why. `df` after versus before plus per-item accounting; explain deltas (shared store closures, GC over- or under-estimate). List removed (sizes), relocated (old→new), kept and why, manifest and lock changes, declined GC with estimated yield, and any assumption the numbers proved wrong.
