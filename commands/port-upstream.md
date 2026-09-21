---
description: Prepare, submit and track an upstream contribution from port work (linux-phone-porting)
argument-hint: "[patch, series, capability, or existing submission]"
---

Prepare or resume an upstream contribution for: $ARGUMENTS

**First: refresh the session's task list** — replace whatever is there with this command's steps, one item per step, first step in progress (`SKILL.md` chain rule).

This command is preparation-first and **human-mediated**: the agent researches, drafts and verifies; **the user owns all maintainer interaction** — submitting, commenting, replying on issues/PRs/mailing lists. Default output is a review-ready bundle plus suggested messages, never a posted issue, PR, email or comment. Any agent-assisted publication needs a separate explicit request approving that exact action and content. This command never touches the device.

**You are responsible for submitting and talking to maintainers.** Review technical claims, public identity, sign-offs and attachments before sending. **Bring replies, review comments and CI results back** — they change the implementation, test requirements, destination or porting approach, and are worth more than a clean local record.

## 1. Check existing work — repeat before every submission and revision

Search for the capability/fix across: owning upstream code and history, development trees, pending series/PRs/issues, downstream ports, sister-device implementations and the component inventory's document records. Distinguish **merged / under review / working downstream / similar-but-inapplicable**. Record searched revisions, coverage, access failures — absence within searched sources is not universal absence (phase 2 rule).

- Applicable work exists → **adopting it is success**: recommend adopt/test/backport/contribute-to-effort with applicability evidence.
- Equivalent effort under review → prepare a coordination message for the user; never silently open a competing submission.

## 2. Map destinations and read their rules

Identify every plausible target: relevant kernel subsystems, SoC vendor projects, userspace libraries, firmware-adjacent repos, distro integration. For each, read **current** contribution requirements: target tree/branch, patch format, commit/DCO rules, issue-first policies, review process, security-disclosure policy. Record dependency ordering (bindings before drivers, libraries before users) and cross-project coordination. Distro copies of a fix are not independent destinations — trace upstream.

## 3. Prepare and verify the candidate

Isolated checkout at the destination's stated base. **Remove diagnostic workarounds** (phase 3 minimize rule applies upstream too). One logical concern per patch; document dependencies; preserve authorship/provenance; separate hardware-tested revisions from later rebases — test claims do not transfer across changed content.

Verify: whole series applies to that base; the destination's checks (checkpatch/CI/lint) run and pass; reproduction/evidence redacted of device-unique data, operator-supplied secrets and identifiers — Wi-Fi SSIDs/passwords, accounts, tokens, IMEI/serial/MAC, calibration, local hostnames (the phase 0 publication-hygiene rule) — verified by scanning the payload itself. Produce commit messages, cover letter/PR body and the evidence bundle with claimed boundaries explicit.

## 4. Submission gate — hand off, do not publish

Present: exact payload (patches/body), destination, recipients/maintainers, public identity, sign-offs and whether the tested revision matches the candidate. Missing credentials or unconfirmed identity ⇒ **prepared, not submitted**. Verify the send path before the first real submission: stale app passwords and MUA integrations have produced silent non-delivery and accidental duplicate copies — a self-test send is cheap, and duplicates that escaped get a short disclaimer naming the operative Message-ID. When an archive (Patchwork, lore) is unreachable from this host, ground thread state in an accessible one (marc.info, project API) and record which grounded each state. Never invent review tags, maintainer consent or DCO certification; a reviewed-by applies only to content-identical patches.

Record in the submission ledger (project's existing upstream format): destination + base revision + rules snapshot; candidate revision + tested identity; evidence + limitations; thread/PR IDs + feedback + response status; **state ∈ prepared/submitted/accepted/merged/released** — distinct, each with evidence — plus one next action.

## 5. Review, revisions, landing

Resume from the recorded thread. Map each comment: change → revision with rerun checks; disagreement → evidence-backed draft reply for the user; unresolved → named question. Revisions carry what changed and which tags survive. Re-check existing-work rule before resubmitting. **Merged ≠ released ≠ deployed**: retire the local patch only after verifying the fix in the version the port actually consumes (phase 3 pristine-battery rule). Maintainer silence is a paused state, not acceptance — report status honestly and stop. First-time-contributor limits — hidden CI logs, reviewer assignment — are operator actions per CONTRIBUTING; record them in the ledger as operator-owned next actions.

**Resubmission mechanics (vN).** Regenerate from scratch: a fresh repo seeded with upstream-master files plus `git am` of the _posted_ prior-version mboxes — the diff base then matches what reviewers saw. Thread the v2 cover letter In-Reply-To the prior cover. Carry review tags (R/A) only on content-identical patches; drop them where content changed. Attribute residual checkpatch warnings to a trimmed local tree missing `Documentation/` before treating them as findings. A submission directory is self-contained: cover letter + patches + a README stating status + the send script.

Report: disposition per §1 (adopt/coordinate/prepare/blocked/review), ledger state, exact next action and its owner.
