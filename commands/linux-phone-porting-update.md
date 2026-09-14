---
description: Update an installed linux-phone-porting skill and its companion commands without discarding local work (maintenance outside the hardware phases)
argument-hint: "[installed skill path or installation scope, if needed]"
---

Update the installed skill, not the device project: $ARGUMENTS

**Stop conditions (apply everywhere).**

- Only run when the operator asked. Never mid-`/port-loop` or mid-experiment.
- Never touch device repos, kernels, images, the phone.
- Any blocker (no network, no permissions, unknown provenance, no safe fast-forward) ⇒ stop and report the manual action. Never improvise, never install tools, never elevate.
- Never: reset, stash, clean, force-checkout, force-fetch, rebase, overwrite a copy, replace an existing link, update-all.

1. **Locate the installation.**
   - Start from the harness's loaded-skill location + configured skill/command dirs.
   - `realpath` the skill-dir and `SKILL.md` links; resolve command links too. Record all paths.
   - Several candidates ⇒ ask the operator to pick one.
   - Detect mechanism from links + metadata, never the dir name:
     - Manual install = Git clone + per-command symlinks into that clone.
     - Manager cache (even with `.git`) = manager-owned.

2. **Pick the path.**
   - **Manager-owned:** use only its documented package-scoped update. Read manifest/lock/receipt + docs first. Preserve channel + local changes. Inspect version + changelog before applying. Afterwards verify skill + commands are the same release.
   - **Copy / unknown / mixed:** do not overwrite. Propose a migration (backup → fresh clone of `https://github.com/angelwzr/linux-phone-porting` → diff → agree channel → operator approval → swap links). Ask before executing.
   - **Verified manual clone:** continue to step 3.

3. **Check identity + local state.** Run everything as `git -C "$repo"`.
   - Verify: `rev-parse --show-toplevel` = that root; `SKILL.md` + `commands/` tracked there.
   - Discover (never guess):

     ```sh
     git -C "$repo" symbolic-ref --quiet --short HEAD
     git -C "$repo" rev-parse --verify 'HEAD^{commit}'
     git -C "$repo" rev-parse --symbolic-full-name '@{upstream}'
     git -C "$repo" config --get-all "branch.$branch.remote"
     git -C "$repo" config --get-all "branch.$branch.merge"
     git -C "$repo" remote get-url --all "$remote"
     git -C "$repo" status --porcelain=v1 --untracked-files=all --ignore-submodules=none
     ```

   - Require: attached branch, one remote, one upstream `refs/heads/…`.
   - Reject: local-path remote, multiple fetch URLs, ambiguous tracking, upstream `.`, `origin/main` assumed, detached/tag-pinned "latest".
   - Fetch URL (after rewrites) must identify `github.com/angelwzr/linux-phone-porting` (HTTPS/SSH/`.git` equivalents). Fork/mirror ⇒ ask about provenance.
   - Status must be empty; no merge/rebase/cherry-pick/revert running; history not shallow.
   - Record full old commit + branch. Local edits: list paths, leave alone.

4. **Fetch + inspect the candidate.**

   ```sh
   git -C "$repo" fetch --no-tags --no-prune --no-recurse-submodules --refmap= "$remote" "$remote_ref"
   git -C "$repo" rev-parse --verify 'FETCH_HEAD^{commit}'
   git -C "$repo" merge-base --is-ancestor "$old" "$new"   # must exit 0, else stop
   git -C "$repo" log --oneline "$old..$new"
   git -C "$repo" diff --stat "$old" "$new"
   git -C "$repo" diff "$old" "$new" -- CHANGELOG.md
   ```

   - `remote_ref` = the single `refs/heads/…` from step 3.
   - Fetch fail ⇒ stop. Pin the fetched ID as `new`; never a moving ref; never `pull`.
   - Show old/new IDs, remote, channel, changelog before advancing.
   - Changelog missing/unchanged ⇒ say so + summarize real diffs.
   - Same ID ⇒ no update; still check links (step 5).

5. **Preflight links + command set.**
   - Diff old/new `commands/` lists at those revisions. Plain Markdown only; no links outside the repo.
   - Resolve every companion-command entry in the selected dirs, including dangling symlinks.
   - Links must target this checkout. Copies or links elsewhere ⇒ block.
   - Check old + new names so no stale command shadows the set.
   - Vacant destination = no entry AND no dangling symlink. Plan a link for each missing command in dirs already exposing this bundle.
   - Removed/renamed commands: retire only exact verified bundle-owned links, with separate approval.

6. **Fast-forward.**
   - Recheck paths, remote/channel, branch, old HEAD, status, links. Anything changed, or another updater running ⇒ stop.

     ```sh
     git -C "$repo" merge --ff-only --no-autostash --no-overwrite-ignore "$new"
     ```

   - Any flag fails ⇒ stop. Never retry with weaker options.
   - Add missing links: plain `ln -s`, full destination paths, never `ln -sf`; recheck vacancy first. Links are a separate step and can fail independently.

7. **Verify + report honestly.**
   - Confirm: HEAD = `new`, clean worktree, `SKILL.md` resolves here, every command resolves to its file at that revision.
   - Git moved but links failed ⇒ report **partial update** + actual HEAD + incomplete destinations. Never claim success, never reset to hide.
   - Couldn't fetch ⇒ report "update not checked", never "up to date".
   - Tell the operator to restart the session. Never resume the hardware loop on mixed old/new instructions.

Refs: [fetch](https://git-scm.com/docs/git-fetch) · [remote](https://git-scm.com/docs/git-remote) · [status](https://git-scm.com/docs/git-status) · [merge-base](https://git-scm.com/docs/git-merge-base) · [merge](https://git-scm.com/docs/git-merge)
