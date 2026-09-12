---
description: Update an installed linux-phone-porting skill and its companion commands without discarding local work (maintenance outside the hardware phases)
argument-hint: "[installed skill path or installation scope, if needed]"
---

Update the installed skill, not the device project: $ARGUMENTS

**Precondition.** The operator explicitly requested maintenance. This command is outside phases 0–3: never invoke it automatically from `/port-loop`, during a flash, or halfway through a hardware experiment. Finish recording the experiment and stop the hardware loop first. Do not change device repositories, project rules, kernels, images or the phone. A directory named `linux-phone-porting` in the current working directory is not proof of an installation.

**Safety contract.** Preserve all local content. Never reset, stash, clean, force a checkout or fetch, rebase, overwrite a copied file, or replace an existing link to make an update work. Never use an update-all operation. Missing network access, permissions, provenance or a safe fast-forward is a blocked update, not permission to improvise. Do not install tools or elevate privileges to bypass a block.

1. **Locate the active installation.** Use the harness's loaded-skill location and configured skill/command directories; inspect user-wide and project-local precedence only within that scope. Follow the skill-directory link and any `SKILL.md` link to their resolved targets with `realpath` or the platform equivalent. Record the visible and resolved paths. Resolve the installed command links too, including this command; an updater from another checkout does not identify the installed skill. If several installations could be active, report them and have the operator select the scope before writing. Do not sweep every project or update every copy.

   Determine the install mechanism from links and actual metadata, not its directory name. The README's manual install is a Git clone exposed directly or through a skill-directory symlink, with individual companion command symlinks into the same clone. A manager-owned cache is **not** a manual clone even if it contains `.git`.

2. **Take the correct maintenance path.**
   - **Manager-owned:** inspect its manifest/lock/receipt and installed manager help or current primary documentation. Discover the exact package identity, source, installed revision and selected version/channel. Use only its documented package-scoped update, preserving that channel and local modifications. Inspect the proposed version and changelog before applying; require a documented way to select that inspected version. Do not invent a manager CLI or run Git inside its cache. If scope, exact-version selection or preservation cannot be established, stop with the specific manual action required. Afterwards verify the skill and companion commands belong to the same installed release; manager success alone is not proof.
   - **Copy, unknown provenance or mixed copies and links:** do not overwrite. Give an actionable migration: preserve the existing skill and command files in an operator-chosen backup outside all harness discovery directories; clone `https://github.com/angelwzr/linux-phone-porting` into a new, unused directory; inspect its revision, changelog and differences from the preserved files; agree the intended branch/channel; then, with separate operator approval, move the old entries to that backup and create the README's links only at now-vacant destinations. Migrate the skill and all selected companion commands together. Keep customizations in a separate development clone. Show exact discovered paths and collisions in the proposal; do not perform this migration as an automatic update.
   - **Verified manual Git clone:** continue below. Commands are optional; an installation with no command directory remains skill-only unless the operator requests commands.

3. **Establish identity and local state before fetching.** Run Git with `-C` against the resolved installation root, never implicitly against the current directory. Check `git rev-parse --show-toplevel` identifies that root and that `SKILL.md` and `commands/` are tracked there, not merely inside an unrelated parent repository. Inspect these values (replace the quoted variables with the discovered values; do not guess them):

   ```sh
   git -C "$repo" symbolic-ref --quiet --short HEAD
   git -C "$repo" rev-parse --verify 'HEAD^{commit}'
   git -C "$repo" rev-parse --symbolic-full-name '@{upstream}'
   git -C "$repo" config --get-all "branch.$branch.remote"
   git -C "$repo" config --get-all "branch.$branch.merge"
   git -C "$repo" remote get-url --all "$remote"
   git -C "$repo" status --porcelain=v1 --untracked-files=all --ignore-submodules=none
   ```

   Require an attached branch, exactly one configured remote and one upstream `refs/heads/…` branch. Preserve that tracking channel; do not assume `origin/main`, switch branches or resolve a detached/tag-pinned checkout to “latest.” Reject local-path remotes, multiple fetch URLs, ambiguous tracking and an upstream remote of `.`. The effective fetch URL (including Git's URL rewrites) must identify `github.com/angelwzr/linux-phone-porting`, allowing equivalent HTTPS/SSH spellings and an optional `.git` suffix. A fork or mirror needs an explicitly agreed provenance/channel decision, not a silent remote rewrite. Check the skill's frontmatter identity and repository layout as well: matching a remote string alone is insufficient.

   Require empty status output, including staged, unstaged and untracked files; also inspect normal status for an operation in progress and stop for a merge, rebase, cherry-pick or revert. Stop on a shallow/incomplete history if ancestry cannot be established. Record the full old commit ID and branch. Local edits are the operator's work: list the blocking paths and leave them untouched. Do not infer that a clean tree means there are no local commits; check history after fetching.

4. **Fetch first, then inspect one immutable candidate.** Fetch only the configured remote branch, without pruning, tags, recursive submodules or configured ref mappings:

   ```sh
   git -C "$repo" fetch --no-tags --no-prune --no-recurse-submodules --refmap= "$remote" "$remote_ref"
   git -C "$repo" rev-parse --verify 'FETCH_HEAD^{commit}'
   ```

   Here `remote_ref` is the single full `refs/heads/…` value discovered above. Stop if fetch fails; a previous `FETCH_HEAD` or remote-tracking ref is not fresh evidence. Immediately retain the successful fetch's full commit ID as `new`; use that ID thereafter, never a moving ref. This fetch downloads the candidate without advancing the installed branch. Do not run `pull` after inspection: it would fetch again.

   ```sh
   git -C "$repo" merge-base --is-ancestor "$old" "$new"
   git -C "$repo" log --oneline "$old..$new"
   git -C "$repo" diff --stat "$old" "$new"
   git -C "$repo" diff "$old" "$new" -- CHANGELOG.md
   git -C "$repo" show "$new:CHANGELOG.md"
   ```

   The ancestry check must exit 0: exit 1 means local-only commits, divergence or rewritten history; any other nonzero result is an error. Stop in either case, leaving local history intact. Inspect and show the old/new IDs, remote, channel and relevant changelog entries to the operator before advancing. If the changelog is missing or unchanged, say so and summarize the actual commit/file differences instead of inventing release notes. Equal IDs mean no revision update, but still check command links before claiming the installation is current.

5. **Preflight the whole exposed installation.** Inspect `SKILL.md` and the old/new `commands/` file lists at those exact revisions, plus their diff. Require the expected skill identity and plain Markdown command files; do not follow candidate links outside the repository. Inventory the selected harness command directories and resolve each known companion command entry, including dangling symlinks. Existing links must point to the corresponding file in this same checkout; regular-file copies or links elsewhere block the update until separately reconciled. Check both old and new command names so a stale command cannot quietly shadow the updated set.

   Plan a link for every missing companion command in each selected directory already exposing this bundle, including newly introduced commands. A directory symlink to this checkout's `commands/` needs no per-file links. A destination is vacant only if neither a filesystem entry **nor a dangling symlink** exists. Preflight permissions and collisions before moving the branch. If an old command is removed or renamed, identify only its exact, verified bundle-owned links and ask for separate approval to retire those entries; leave unrelated commands alone. Never bulk-delete command symlinks. Do not advance while this would leave an unresolved mixed or stale command set.

6. **Fast-forward only the inspected revision.** Immediately before applying, recheck the resolved installation paths, remote/channel, branch, old HEAD, clean status and link preflight; stop if anything changed or another updater is running. Then run:

   ```sh
   git -C "$repo" merge --ff-only --no-autostash --no-overwrite-ignore "$new"
   ```

   `--ff-only` forbids a merge commit; `--no-autostash` prevents configuration from hiding edits; `--no-overwrite-ignore` protects ignored files that collide with incoming tracked files. If any safeguard fails, stop rather than retrying with weaker options. Add the preflighted missing links with plain `ln -s` to explicit full destination paths, never `ln -sf`; recheck vacancy just before creation. Existing correct links need no replacement. Updating the shared checkout updates the skill and existing command targets together; adding command links is a separate step that can fail.

7. **Verify and report the actual outcome.** Confirm HEAD equals the inspected `new` ID, the worktree is clean, the installed `SKILL.md` resolves to this checkout, and every selected companion command resolves to its corresponding file at that same revision. Report old/new revisions, channel, affected install paths, added links and any preserved blockers. If Git advanced but link creation or verification failed, report **partial update**, the actual current HEAD and each incomplete destination; do not claim success or reset to hide the failure. Give a narrowly scoped next action that preserves existing entries. If fetching was unavailable, report “update not checked,” not “up to date.”

   Ask the operator to restart the agent session after an update so loaded skill/command text is refreshed. Do not resume the hardware loop using a mixture of old instructions in memory and new files on disk.

**Git reference.** These safeguards use the documented behavior of [fetch and explicit ref mappings](https://git-scm.com/docs/git-fetch), [remote URL resolution](https://git-scm.com/docs/git-remote), [status including untracked files](https://git-scm.com/docs/git-status), [ancestry checks](https://git-scm.com/docs/git-merge-base), and [fast-forward merge and preservation options](https://git-scm.com/docs/git-merge).
