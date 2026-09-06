# linux-phone-porting

An agent skill for porting mainline Linux to a phone. It makes your coding agent gather evidence and do research **before** it writes a fix.

## Why

Every guess on a phone under bring-up costs a build, a flash, and a boot. Five to ten minutes, and most guesses are wrong.

So the skill imposes an order on the session:

1. Back up the stock system while it is still readable — it is also your best research source.
2. Capture evidence from the live device.
3. Research the problem across an ordered, complete list of independent source families.
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

| Phase            | What it enforces                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **0. Set up**    | Unlocked bootloader required. Pin the exact variant, target distro, shell and boot topology, back up every partition but userdata, health-check the storage, inventory the components against the official specs, harvest everything the stock system offers — then mine it all; the stock system is the best research source you get.                                                                                                                                                                        |
| **1. Evidence**  | Debug knobs on before you reproduce. Full dmesg. pstore from the path systemd actually leaves it in. Hash what is really flashed instead of trusting your notes.                                                                                                                                                                                                                                                                                                                                              |
| **2. Research**  | Every source family, every time — whether you are chasing a crash log or building something new. The first plausible hit is not the answer. It also sweeps sister devices — same-family models that already have ports — diffing their board files component-by-component against your phase 0 inventory: a match is a proven combination, never a precedent. An empty sweep routes to the primary artefacts — vendor driver, stock DTB, closest mainline sibling — and the design gets derived, not shelved. |
| **3. Implement** | One variable per flash, hypothesis stated up front, cheap runtime tests over reflashes, and the pre-build battery — patches apply, symbols exist, config options exist, touched units compile — before any full rebuild. Three refutations means your model is wrong: go back to phase 2 — the rule stops the guessing, not the work.                                                                                                                                                                         |

## Commands

Optional slash commands, one per phase, for when you want a single step rather than the whole workflow.

- **`/port-setup`** — _run once, at the start of a port._ Pins down the exact device variant, target distro, shell and boot topology, takes the pre-flash backup, health-checks the storage and inventories the components against the official specs, then extracts the stock DTB, firmware layout and vendor configs. Everything later phases research against comes from here.

- **`/evidence-sweep`** — _the device just failed._ Turns on the debug knobs, reproduces, and captures everything: dmesg, pstore, console records, a read-back hash of what is actually flashed. Deliberately refuses to diagnose — the output is evidence, nothing more.

- **`/source-sweep`** — _you have evidence and need answers._ Fans out across every source family (mainline, the SoC vendor's mainline project, vendor kernel, postmarketOS, Halium/UBports, Mobian, NixOS) and sweeps sister devices for family precedent, then reports what each one said, including the ones that said nothing. Ends with hypotheses ranked by what backs them.

- **`/port-research`** — _you have been asked to build something that has never worked._ The same sweep entered from the other side: no crash log, so the device data is the phase 0 material. Ends with a plan and its sources attached, not a hypothesis about a symptom.

- **`/flash-gate`** — _you have a fix and it is about to cost a boot._ Six questions: what is the hypothesis, which source backs it, is it one variable, can it be tested without flashing, how many fixes have already been refuted, and what result would refute this one. Any "no" blocks the flash. If the fix needs a full rebuild, the pre-build battery (patches apply, symbols exist, config options exist, touched units compile) runs first.

- **`/port-loop`** — _you want a capability working and are happy to let the agent iterate._ Asks how many refutations you allow (5, 15, or unlimited), then runs the loop itself: research, gate, apply the cheapest test available, verify, record what the result rules out. It flashes boot, dtbo and modules on its own once the gate passes, and stops for you before the bootloader, modem NV/EFS, persist or the partition table — anything the phase 0 backup cannot undo. Every third refutation on the same symptom forces a scope widening, not a stop; the loop hands back when your budget is spent, the device stops answering, or nothing testable remains.

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

- **2026-09-06**
  - `/port-loop` no longer dies at three refutations. At invocation it asks the operator for a refutation budget — 5, 15, or unlimited — recorded in the ledger header; every third refutation on the same symptom forces a scope widening (the layer above, or the primary artefacts) rather than a stop, and the loop hands back only when a finite budget is spent, the device stops answering, or nothing testable remains. `/flash-gate`'s refutation question reads against that budget when running under the loop.
  - `/port-setup` ends by locking the project to the ruleset: it writes an explicit rule into the project's agent context files (`CLAUDE.md`, `AGENTS.md`, equivalents) that all work touching the device goes through the skill's phases and gates — no device action outside the ruleset.
  - Phase 0 sets the project layout, proves the control channels before any flash, decodes A/B slot state from the on-disk structure, and requires a media-failure verdict to survive a later-session re-probe before it retires a device.
  - Phase 1: the kernel ACM console must be verified against its reporting context (and tested with a crash injector) before spending a flash; ramoops rot includes warm reboots; unchanging status registers are latches until proven live; channel loss after suspend/resume is three-valued.
  - Phase 2 gains a sister-device sweep, and reads DSP silence as a possibly unanswered callback; patches are re-verified against every downstream DT held.

Full history in [CHANGELOG.md](CHANGELOG.md).

## License

[CC BY 4.0](LICENSE) © [Roman Linev](https://rlinev.ru).
