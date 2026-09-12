# linux-phone-porting

An agent skill for porting mainline Linux to a phone. It makes your coding agent gather evidence and do research **before** it writes a fix.

## Why

Every guess on a phone under bring-up costs a build, a flash, and a boot. Five to ten minutes, and most guesses are wrong.

So the skill imposes an order on the session:

1. Back up the stock system while it is still readable — it is also your best research source.
2. Capture evidence from the live device.
3. Research the problem across an ordered, complete list of independent source families.
4. Only then write a fix, one variable at a time.

The skill originated in one real port — mainline Linux on a Xiaomi Mi A3, running NixOS on kernels 7.1 and 7.2 — and now incorporates lessons from other ports while keeping each measurement's scope explicit. The original hardware established which evidence channels survive a wedge and which quietly lose your logs, which observations manufacture the very symptom they are meant to measure, and why the stock DTB pulled off the device beats the sibling SoC's device tree whenever the two disagree. Its evidence notes keep the shape that external review worked against — numbered eliminations, an honest-limits section, and a method block exact enough to repeat — and an outside maintainer's review of those notes, taken as tracked changes, has caught real risks. Transfer the method, not a claim that every device produced the same result.

## Install

**Easiest — ask your agent.** Paste this into Claude Code (or any agent with shell access):

> Install the skill from https://github.com/angelwzr/linux-phone-porting — clone it into an unused `~/.agents/skills/linux-phone-porting`, symlink it into my skills directory, and symlink `commands/*.md` into my commands directory. Inspect existing entries first; preserve any copies or conflicting links rather than overwriting them.

**By hand.** Clone once, link wherever you need it:

```sh
git clone https://github.com/angelwzr/linux-phone-porting ~/.agents/skills/linux-phone-porting

# Claude Code, user-wide
mkdir -p ~/.claude/skills ~/.claude/commands
ln -s ~/.agents/skills/linux-phone-porting ~/.claude/skills/linux-phone-porting
ln -s ~/.agents/skills/linux-phone-porting/commands/*.md ~/.claude/commands/
```

For a single project instead of user-wide, link into `<project>/.claude/skills/` and `<project>/.claude/commands/`.

These commands are for a fresh installation: the clone destination and link names must be unused. Do not add `-f` to get past a collision. Keep one checkout for the skill and all its companion commands, and keep development edits in a separate clone.

The skill is plain markdown with standard frontmatter, so it works in any harness that reads skills. Point that harness at the same clone — one checkout, exposed through links (see [Updating](#updating)). The commands are optional; the skill works without them.

## Updating

Run **`/linux-phone-porting-update`** between hardware sessions. It resolves the installed skill and command links rather than updating whichever development checkout happens to be the current directory. For the manual install above, it verifies the repository and tracking branch, requires a clean worktree with no untracked files or local-only commits, fetches the candidate, shows the old/new revisions and changelog, then fast-forwards to that exact inspected revision. It does not switch channels or discard local work.

Existing links follow the shared checkout, but **new command files need new links**: the install-time wildcard is not a live subscription. The updater adds missing companion links only at vacant destinations in the selected command directories. Copies, dangling links and links into another checkout are reported rather than overwritten. Skill and commands must come from the same revision; a link failure after Git advances is reported as a partial update.

**Bootstrapping an older installation.** If `/linux-phone-porting-update` is not yet discoverable, ask your agent:

> Read the current [linux-phone-porting updater instructions](https://github.com/angelwzr/linux-phone-porting/blob/main/commands/linux-phone-porting-update.md) and carry out that maintenance procedure against my installed skill. Resolve the installed paths first, inspect the fetched revision and changelog, preserve all local work, and add the missing updater and companion command links only at vacant destinations. Do not update a development clone or touch the phone.

This lets the first safe update add the updater link; restart the agent session afterwards to load the new command and skill text. Do not copy only the updater into an old bundle or rerun the wildcard link command with force.

Keep the installed checkout **pull-only**. If it contains local edits, untracked files or local commits, stop and preserve them; review or migrate that work into a separate development clone before retrying. Never reset, stash, force or overwrite to make an update pass. A manager-owned installation uses its discovered, package-scoped update mechanism, not Git in its cache or an update-all command. For manual copies or unknown provenance, the updater proposes a backed-up migration to a fresh clone and links, with separate approval before replacing any installed entries. Network or permission failures leave the update explicitly unverified.

## Requirements

A phone with an **unlocked bootloader**. Locked devices are out of scope — the skill does not unlock bootloaders and does not advise on unlocking. Unlocking wipes data on most devices and one wrong step can brick the phone, so unlock first, through your device maker's own instructions, before starting a port.

## The four phases

| Phase            | What it enforces                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **0. Set up**    | Unlocked bootloader required. Pin the exact variant, target distro, shell and boot topology, back up every partition but userdata, health-check the storage, inventory the components against the official specs, harvest everything the stock system offers — then mine it all; the stock system is the best research source you get.                                                                                                                                                                        |
| **1. Evidence**  | Debug knobs on before you reproduce. Full dmesg. pstore from the path systemd actually leaves it in. Hash what is really flashed instead of trusting your notes.                                                                                                                                                                                                                                                                                                                                              |
| **2. Research**  | Every source family, every time — whether you are chasing a crash log or building something new. The first plausible hit is not the answer. It also sweeps sister devices — same-family models that already have ports — diffing their board files component-by-component against your phase 0 inventory: a match is a proven combination, never a precedent. An empty sweep routes to the primary artefacts — vendor driver, stock DTB, closest mainline sibling — and the design gets derived, not shelved. |
| **3. Implement** | One variable per flash, hypothesis stated up front, cheap runtime tests over reflashes, and the pre-build battery — patches apply, symbols exist, config options exist, touched units compile — before any full rebuild. Three refutations means your model is wrong: go back to phase 2 — the rule stops the guessing, not the work.                                                                                                                                                                         |

## Commands

Optional slash commands for individual phases, the full loop, and separate installation maintenance.

- **`/port-setup`** — _run once, at the start of a port._ Pins down the exact device variant, target distro, shell and boot topology, takes the pre-flash backup, health-checks the storage and inventories the components against the official specs, then extracts the stock DTB, firmware layout and vendor configs. Everything later phases research against comes from here.

- **`/evidence-sweep`** — _the device just failed._ Turns on the debug knobs, reproduces, and captures everything: dmesg, pstore, console records, a read-back hash of what is actually flashed. Deliberately refuses to diagnose — the output is evidence, nothing more.

- **`/source-sweep`** — _you have evidence and need answers._ Fans out across every source family (mainline, the SoC vendor's mainline project, vendor kernel, AOSP, postmarketOS, Halium/UBports, Mobian, NixOS) and sweeps sister devices for family precedent, then reports what each one said, including the ones that said nothing. The ordered source list in `SKILL.md` is authoritative. Ends with hypotheses ranked by what backs them.

- **`/port-research`** — _you have been asked to build something that has never worked._ The same sweep entered from the other side: no crash log, so the device data is the phase 0 material. Ends with a plan and its sources attached, not a hypothesis about a symptom.

- **`/flash-gate`** — _you have a fix and it is about to cost a boot._ Six questions: what is the hypothesis, which source backs it, is it one variable, can it be tested without flashing, how many fixes have already been refuted, and what result would refute this one. Any "no" blocks the flash. If the fix needs a full rebuild, the pre-build battery (patches apply, symbols exist, config options exist, touched units compile) runs first.

- **`/port-loop`** — _you want a capability working and are happy to let the agent iterate._ Asks how many refutations you allow (5, 15, or unlimited), then runs the loop itself: research, gate, apply the cheapest test available, verify, record what the result rules out. It flashes boot, dtbo and modules on its own once the gate passes, and stops for you before the bootloader, modem NV/EFS, persist or the partition table — anything the phase 0 backup cannot undo. Every third refutation on the same symptom forces a scope widening, not a stop; the loop hands back when your budget is spent, the device stops answering, or nothing testable remains.

- **`/linux-phone-porting-update`** — _you explicitly want to maintain the installed skill._ Verifies install provenance and channel, reviews the fetched changelog, fast-forwards a clean Git installation to the inspected revision, and reconciles companion links without overwriting local work. Runs outside the hardware phases; never self-updates from `/port-loop`.

`/source-sweep` and `/port-research` are the same phase approached from opposite directions — one starts from a symptom, the other from a request — and both feed `/flash-gate`. Each command states its own precondition, so you can start anywhere.

## Pairs well with

The skill invokes companion skills **by name**, each with a plain-tools fallback when it is not installed. Nothing breaks if you install none of them — the skill works end-to-end on base capabilities — but research and source navigation get better with them:

- **`linux-kernel-development`** (fallback: `linux-kernel-crash-debug`, then any subsystem-specific kernel skill) — to work out which driver, binding or firmware interface owns a failure. Without any of them, the agent reasons through the driver sources directly.

- **`find-docs`** (Context7-backed) — current API and configuration details on the userspace libraries involved, rather than recalled signatures. Without it, the agent reads the project's docs over the web.

- **`wigolo`** — for the phase 2 sweep; its local cache is the part that matters, since consecutive sessions re-read the same handful of pages. Without it, the agent uses plain web search.

- **`graft`** — optional navigation for an indexed source tree: locate symbols, callers and relevant spans before broad searches. The exact checked-out source remains authoritative; an index or summary is a pointer, not patch evidence. Without it, use direct source search and reads.

Any skill covering the same capability can substitute — the names are the defaults the skill tries first, not hard dependencies.

## Adapting it

The skill is deliberately device-agnostic: no tool paths, no partition names, no hashes. Keep your own port's specifics in your project's `CLAUDE.md` or a sibling skill, and leave this one as the method.

## Changelog

- **2026-09-12**
  - **Phase 0 — setup:** layered source/artifact layout, scoped private material, exact product identity and transports, operation-specific boot/fetch checks, verified recovery, and separate declared, bound and exercised capability claims. Transport binding now also constrains the host: reachability probes never mutate host network state, and a ping is not target identity.
  - **Phase 1 — evidence:** acquisition integrity and coverage, exact deployed identity, and functional readiness separated from acoustic qualification.
  - **Phase 2 — research:** AOSP and OEM/ODM aliases and archives, exact-source security evidence and portable reproductions, plus optional `graft` navigation.
  - **Phase 3 — implementation:** pristine ordered patch checks, inert moves separated from upgrades, complete image assembly, real-reboot acceptance and rollback-preserving measured cleanup. Adds per-artifact build scoping and status, transactional fail-closed source preparation, deployment as a state machine with separately observed postconditions, and FAIL-never-SKIP revalidation of required probes.
  - **Maintenance:** `/linux-phone-porting-update` and safe installation migration; no destructive reset advice. Details and measurement caveats are in the [full changelog](CHANGELOG.md).

Full history in [CHANGELOG.md](CHANGELOG.md).

## License

[CC BY 4.0](LICENSE) © [Roman Linev](https://rlinev.ru).
