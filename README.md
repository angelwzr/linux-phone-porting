# linux-phone-porting

An agent skill for porting mainline Linux to a phone. It makes your coding agent gather evidence and do research **before** it writes a fix.

## Why

Every guess on a phone under bring-up costs a build, a flash, and a boot. Five to ten minutes, and most guesses are wrong.

So the skill imposes an order on the session:

1. Back up the stock system while it is still readable — it is also your best research source.
2. Capture evidence from the live device.
3. Research the problem across seven independent sources.
4. Only then write a fix, one variable at a time.

Everything in it came out of one real port — mainline Linux on a Xiaomi Mi A3, running NixOS on kernels 7.1 and 7.2 — and every claim was measured on that hardware: which evidence channels survive a wedge and which quietly lose your logs, which observations manufacture the very symptom they are meant to measure, and why the stock DTB pulled off the device beats the sibling SoC's device tree whenever the two disagree. Its evidence notes keep the shape that external review worked against — numbered eliminations, an honest-limits section, and a method block exact enough to repeat — and an outside maintainer's review of those notes, taken as tracked changes, has caught real risks.

## Install

**Easiest — ask your agent.** Paste this into Claude Code (or any agent with shell access):

> Install the skill from https://github.com/angelwzr/linux-phone-porting — clone it into `~/.agents/skills/linux-phone-porting`, symlink it into my skills directory, and symlink `commands/*.md` into my commands directory.

**By hand.** Clone once, link wherever you need it:

```sh
git clone https://github.com/angelwzr/linux-phone-porting ~/.agents/skills/linux-phone-porting

# Claude Code, user-wide
mkdir -p ~/.claude/skills ~/.claude/commands
ln -s ~/.agents/skills/linux-phone-porting ~/.claude/skills/linux-phone-porting
ln -s ~/.agents/skills/linux-phone-porting/commands/*.md ~/.claude/commands/
```

For a single project instead of user-wide, link into `<project>/.claude/skills/` and `<project>/.claude/commands/`.

The skill is plain markdown with standard frontmatter, so it works in any harness that reads skills. Point that harness at the same clone — one checkout, and updates are one pull (see [Updating](#updating)). The commands are optional; the skill works without them.

## Updating

One checkout, one pull. Everything your harness sees — the skill and the slash commands — is a symlink into the checkout, so a single command updates all of it at once:

```sh
git -C ~/.agents/skills/linux-phone-porting pull --ff-only
```

`--ff-only` makes a diverged history fail loudly instead of silently merging. Project-local links point at the same checkout and follow the same pull.

Keep the checkout **pull-only**. Make changes in your own clone or fork, never through a symlinked path — if `git -C ~/.agents/skills/linux-phone-porting status` shows modifications, someone edited the wrong copy and the next pull will conflict. `git -C ~/.agents/skills/linux-phone-porting reset --hard origin/main` throws local edits away and re-syncs.

## Requirements

A phone with an **unlocked bootloader**. Locked devices are out of scope — the skill does not unlock bootloaders and does not advise on unlocking. Unlocking wipes data on most devices and one wrong step can brick the phone, so unlock first, through your device maker's own instructions, before starting a port.

## The four phases

| Phase            | What it enforces                                                                                                                                                                                                                                                                                                                       |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **0. Set up**    | Unlocked bootloader required. Pin the exact variant, target distro, shell and boot topology, back up every partition but userdata, health-check the storage, inventory the components against the official specs, harvest everything the stock system offers — then mine it all; the stock system is the best research source you get. |
| **1. Evidence**  | Debug knobs on before you reproduce. Full dmesg. pstore from the path systemd actually leaves it in. Hash what is really flashed instead of trusting your notes.                                                                                                                                                                       |
| **2. Research**  | All seven source families, every time — whether you are chasing a crash log or building something new. The first plausible hit is not the answer. An empty sweep routes to the primary artefacts — vendor driver, stock DTB, closest mainline sibling — and the design gets derived, not shelved.                                      |
| **3. Implement** | One variable per flash, hypothesis stated up front, cheap runtime tests over reflashes, and the pre-build battery — patches apply, symbols exist, config options exist, touched units compile — before any full rebuild. Three refutations means your model is wrong: go back to phase 2 — the rule stops the guessing, not the work.  |

## Commands

Optional slash commands, one per phase, for when you want a single step rather than the whole workflow.

- **`/port-setup`** — _run once, at the start of a port._ Pins down the exact device variant, target distro, shell and boot topology, takes the pre-flash backup, health-checks the storage and inventories the components against the official specs, then extracts the stock DTB, firmware layout and vendor configs. Everything later phases research against comes from here.

- **`/evidence-sweep`** — _the device just failed._ Turns on the debug knobs, reproduces, and captures everything: dmesg, pstore, console records, a read-back hash of what is actually flashed. Deliberately refuses to diagnose — the output is evidence, nothing more.

- **`/source-sweep`** — _you have evidence and need answers._ Fans out across all seven sources (mainline, the SoC vendor's mainline project, vendor kernel, postmarketOS, Halium/UBports, Mobian, NixOS) and reports what each one said, including the ones that said nothing. Ends with hypotheses ranked by what backs them.

- **`/port-research`** — _you have been asked to build something that has never worked._ The same sweep entered from the other side: no crash log, so the device data is the phase 0 material. Ends with a plan and its sources attached, not a hypothesis about a symptom.

- **`/flash-gate`** — _you have a fix and it is about to cost a boot._ Six questions: what is the hypothesis, which source backs it, is it one variable, can it be tested without flashing, how many fixes have already been refuted, and what result would refute this one. Any "no" blocks the flash. If the fix needs a full rebuild, the pre-build battery (patches apply, symbols exist, config options exist, touched units compile) runs first.

- **`/port-loop`** — _you want a capability working and are happy to let the agent iterate._ Runs the loop itself: research, gate, apply the cheapest test available, verify, record what the result rules out. It flashes boot, dtbo and modules on its own once the gate passes, and stops for you before the bootloader, modem NV/EFS, persist or the partition table — anything the phase 0 backup cannot undo. It also stops after three refutations (having widened scope once), and when the device stops answering.

`/source-sweep` and `/port-research` are the same phase approached from opposite directions — one starts from a symptom, the other from a request — and both feed `/flash-gate`. Each command states its own precondition, so you can start anywhere.

## Pairs well with

The skill invokes three companion skills **by name**, each with a plain-tools fallback when it is not installed. Nothing breaks if you install none of them — the skill works end-to-end on base capabilities — but the research phase gets noticeably better with them:

- **`linux-kernel-development`** (fallback: `linux-kernel-crash-debug`, then any subsystem-specific kernel skill) — to work out which driver, binding or firmware interface owns a failure. Without any of them, the agent reasons through the driver sources directly.

- **`find-docs`** (Context7-backed) — current API and configuration details on the userspace libraries involved, rather than recalled signatures. Without it, the agent reads the project's docs over the web.

- **`wigolo`** — for the phase 2 sweep; its local cache is the part that matters, since consecutive sessions re-read the same handful of pages. Without it, the agent uses plain web search.

Any skill covering the same capability can substitute — the names are the defaults the skill tries first, not hard dependencies.

## Adapting it

The skill is deliberately device-agnostic: no tool paths, no partition names, no hashes. Keep your own port's specifics in your project's `CLAUDE.md` or a sibling skill, and leave this one as the method.

## Changelog

- **2026-09-05** — Phase 0 now inventories every component against the official spec sheets, records the storage's own wear/lifetime percentage before anything is diagnosed as failing (measured: that reading overturned a false "storage is dying" verdict; the real fault cleared on one more stock-ROM fastboot flash), harvests deeper from rooted stock — treated like bootloader unlocking: the user roots the device, the skill never provides the steps — and records the newest community custom ROM as an optional up-to-date source (measured: an unofficial recent-Android build was the most responsive OS seen on one device). Setup also asks for the target userspace shell instead of assuming one, and strongly recommends the distro's own shell where it ships one. Phase 2 gains a seventh source family — the SoC vendor's mainline collaboration project (e.g. github.com/qualcomm-linux) — and now invokes its companion skills by name (`linux-kernel-development`/`linux-kernel-crash-debug`, `find-docs`, `wigolo`), each with a plain-tools fallback so the skill still works when none is installed. Phase 3 gains a strict pre-rebuild gate — patches apply cleanly, every referenced symbol exists in the tree, every config option exists in the kernel's Kconfig (non-existent options are silently ignored), touched units compile alone before any full kernel rebuild (measured: four consecutive rebuilds burned on missing symbols in unchecked patches) — and an edit-precision rule: changes land by reading the exact current lines and writing the exact replacement; an edit that needs "repairing" afterwards is redone from a fresh read, and self-inflicted breakage then hand-fixed is recorded as a defect, not a workflow.
- **2026-08-31** — Phase 2 names the GPL-published OEM kernel sources as an explicit oracle: phase 0 records the `/proc/version` fingerprint, and the exact-device release gets mined for board dts, defconfig and out-of-tree vendor drivers instead of being assumed buildable.
- **2026-08-30** — Requirements made explicit: an unlocked bootloader is required — locked devices are out of scope and get pointed at the OEM's own unlocking instructions. Setup also records whether a custom recovery is installed.
- **2026-08-24** — New `/port-loop` command: an iterate-until-done orchestrator over the four phases.
- **2026-08-23** — First release: the evidence-first skill for mainline bring-up, with one command per phase.

## License

[CC BY 4.0](LICENSE) © [Roman Linev](https://rlinev.ru).
