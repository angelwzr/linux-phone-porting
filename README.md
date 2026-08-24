# linux-phone-porting

An [agent skill](https://code.claude.com/docs/en/skills) that enforces evidence-first debugging when porting mainline Linux to a phone.

Bring-up iterations are expensive — a build, a flash, and a boot for every guess — and blind fixes get refuted far more often than they land. The skill imposes an order on the session: back up the stock system while it is still readable, capture evidence from the live device, research across mainline, the vendor kernel, postmarketOS, Halium/UBports, Mobian and NixOS, and only then write a fix, one variable per flash.

Everything in it is a lesson from a real mainline port, generalised: why the pre-flash backup is a research source and not just insurance, which evidence channels survive a wedge and which quietly lose your record, the observations that manufacture the symptom they are meant to measure, and why the stock DTB on the device beats the sibling SoC's device tree when the two disagree.

## Install

The skill is plain `SKILL.md` plus markdown commands, so it works in any harness that reads skill frontmatter. Clone it once into a shared location and link it wherever you need it:

```sh
git clone https://github.com/angelwzr/linux-phone-porting ~/.agents/skills/linux-phone-porting
```

Then link it into each harness that should see it:

```sh
# Claude Code — user-wide
mkdir -p ~/.claude/skills
ln -s ~/.agents/skills/linux-phone-porting ~/.claude/skills/linux-phone-porting

# a single project instead of user-wide
ln -s ~/.agents/skills/linux-phone-porting <project>/.claude/skills/linux-phone-porting
```

Harnesses that read `~/.agents/skills/` directly need no link at all. For anything else, point its skill directory at the same clone — one checkout, one `git pull` to update everything.

The commands are optional and install separately:

```sh
mkdir -p ~/.claude/commands
ln -s ~/.agents/skills/linux-phone-porting/commands/*.md ~/.claude/commands/
```

## The method

| Phase            | What it enforces                                                                                                                                                                     |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **0. Set up**    | Pin the exact variant, target distro and boot topology. Back up every partition but userdata _before_ unlocking, and mine it — the stock system is the best research source you get. |
| **1. Evidence**  | Debug knobs on before reproducing, full dmesg, pstore from the path systemd actually leaves it in. Hash what is flashed rather than trusting notes.                                  |
| **2. Research**  | Six source families, all of them, before a line is written — whether the input is a crash log or a request to build something new. The first plausible hit is not the answer.        |
| **3. Implement** | One variable per flash, hypothesis stated up front, cheap runtime tests preferred. Three refutations means the model is wrong — go back to phase 2.                                  |

## Commands

Each runs one phase of the skill on its own, for when you do not want the whole workflow.

| Command           | Phase | Use it when                                                                                                                  |
| ----------------- | ----- | ---------------------------------------------------------------------------------------------------------------------------- |
| `/port-setup`     | 0     | Starting a new port. Pins down the exact variant and target distro, then takes the backup the later phases research against. |
| `/evidence-sweep` | 1     | The device just failed and you want everything captured before anything changes. Produces evidence, refuses to diagnose.     |
| `/source-sweep`   | 2     | Evidence is in hand and you need the six-source research fan-out. Reports per source, including the ones that had nothing.   |
| `/port-research`  | 2     | You have been asked to build or enable something that has never worked. Same sweep, entered from the capability side.        |
| `/flash-gate`     | 3     | You have a fix and want it challenged before it costs a boot. Six questions; any "no" blocks the flash.                      |

`/port-setup` runs once per port. `/source-sweep` and `/port-research` are the same phase from opposite directions — one starts from a symptom, the other from a request — and both feed `/flash-gate`. Each states its precondition rather than assuming the previous one ran.

## Recommended skills to pair with

The skill references three capabilities without hard-depending on any of them — it checks whether one is installed and falls back to plain reasoning if not, so nothing breaks when they are absent. Install them and the research phase gets materially better:

- **A kernel-development skill** — for identifying which driver, binding, or firmware interface owns a failure. `linux-kernel-development` and `linux-kernel-crash-debug` both fill this role; a subsystem-specific one (modules, DRM, remoteproc) helps where it matches.
- **A documentation-retrieval skill** — for current API and configuration details on userspace libraries and tools involved in a bring-up, rather than recalled signatures. `find-docs` (Context7-backed) is one such.
- **A web-research skill** — for the phase 2 source sweep, which is a lot of repeated reading across mailing-list threads, driver sources and distro device repos. `wigolo` is one such; its local cache is the relevant part, since consecutive bring-up sessions tend to re-read the same handful of pages.

Names are given as examples, not as dependencies. Any skill covering the capability works, and the skill degrades to plain reasoning with none installed.

## Adapting it

The skill is deliberately device-agnostic — no tool paths, no partition names, no hashes. If your port has its own harness, keep those specifics in your project's own `CLAUDE.md` or a sibling skill and leave this one as the method.

## License

[CC BY 4.0](LICENSE) © [Roman Linev](https://rlinev.ru).
