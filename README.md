# linux-phone-porting

<p align="center">
  <img src="logo.png" alt="linux-phone-porting logo" width="100%">
</p>

An agent skill for bringing up mainline Linux on phones and tablets. A build, flash, and boot can take 5–10 minutes. Gather evidence before you spend a boot on a guess.

**Contents:** [Why](#why-this-workflow) · [Install](#install) · [Requirements](#requirements) · [Update the skill](#updating) · [Workflow](#the-four-phases) · [Examples](#examples) · [Commands](#commands) · [Workspace](#recommended-folder-structure) · [Achievements](#achievements) · [Changelog](#changelog)

## Why this workflow

- **Save stock evidence before flashing.** The original board description, firmware layout, and vendor configuration tell you how this variant was set up. A sibling phone's device tree can be a useful lead, but it cannot stand in for the board on your bench.
- **Verify what ran before diagnosing it.** A build log tells you what the host produced. Readback and runtime evidence tell you what reached the device and how far it got. If you cannot verify either, report the gap instead of treating an untested change as a failed fix.
- **Test one change before spending another boot.** Write down the result that would disprove the hypothesis. Use a host or runtime check when it can answer the question; save a flash for what actually needs it. Four consecutive rebuilds on one port failed on unchecked missing symbols. The skill now asks you to check patches, symbols, and Kconfig and compile the affected units before a full rebuild.

## Install

Clone to an unused destination:

```sh
git clone https://github.com/angelwzr/linux-phone-porting
```

- If your agent supports skills, register the checkout using its setup instructions.
- Otherwise, give your agent [SKILL.md](SKILL.md), the exact device, and your goal. Attach the file if the agent cannot read your checkout.

Use the same checkout for optional [commands](#commands). Leave existing files and links in place. Use a separate clone to edit the skill.

**Ask your agent to install it:** Paste this into an agent that can run shell commands:

> Clone https://github.com/angelwzr/linux-phone-porting into an unused directory. Register it as a skill using this agent's documented setup. If this agent supports custom slash commands, link the Markdown files in `commands/` from that same checkout. Check existing entries first and stop on a conflict. Tell me which checkout and links you used.

### Claude Code example

This is a fresh, user-wide install on Linux or macOS. [Claude Code supports linked skill directories and Markdown command files](https://code.claude.com/docs/en/skills#where-skills-live). Skip the command loop if you only want the skill.

```sh
(
  set -eu
  src="$HOME/.agents/skills/linux-phone-porting"
  config="$HOME/.claude"

  vacant() {
    if [ -e "$1" ] || [ -L "$1" ]; then
      printf 'Already exists: %s\n' "$1" >&2
      return 1
    fi
  }

  vacant "$src"
  vacant "$config/skills/linux-phone-porting"
  git clone https://github.com/angelwzr/linux-phone-porting "$src"
  mkdir -p "$config/skills" "$config/commands"
  ln -s "$src" "$config/skills/linux-phone-porting"
  for file in "$src"/commands/*.md; do
    link="$config/commands/${file##*/}"
    vacant "$link"
    ln -s "$file" "$link"
  done
)
```

For a project-only install, set `config` to that project's `.claude` directory. If the example stops on an existing entry, inspect it before continuing. Any links already made stay in place.

## Requirements

Start with a demonstrated boot path: an already-unlocked bootloader on retail-unlock hardware, a payload that already boots on exploit-booted hardware, or a proven way to boot the target image on firmware-boot hardware. The skill does not provide unlock or exploit steps.

The skill covers mainline Linux and userspace integration. It does not certify any device or distro combination.

## Updating

Between hardware sessions, run `/linux-phone-porting-update` or give your agent [the updater instructions](commands/linux-phone-porting-update.md) and the installed path. For a manual install, it inspects the candidate revision and changelog before fast-forwarding a clean checkout. It stops rather than overwriting local changes or switching channels.

Existing links follow the shared checkout. A new command file needs a new link; the wildcard used at install time does not add links later. The updater checks directories that already expose the command bundle and skips link reconciliation for a skill-only install. It reports a partial update if Git advanced but the link check failed.

For older installs without the updater, use the [current upstream instructions](https://github.com/angelwzr/linux-phone-porting/blob/main/commands/linux-phone-porting-update.md). Restart the agent session after updating so the skill and commands load from the same revision.

## The four phases

![Four phases: set up the port, capture evidence for a failure, research, then test and verify. A new capability goes from setup to research. Research-only work ends with a host-only gate report. A refuted trial returns to research.](workflow.svg)

### What each phase leaves behind

0. **Setup:** The confirmed device variant, target OS and session, verified backup and recovery path, and an inventory based on stock evidence. Record what the backup covers separately from whether it can be restored.
1. **Evidence:** The failing run's logs, crash records, and readback where available, tied to the observed runtime identity. Record inaccessible channels and unverified fields so the next session knows what is missing.
2. **Research:** A ledger of the sources checked, their revisions or queries, findings that apply, and gaps. It supports ranked hypotheses and names what would disprove each one. If no published fix applies, work from the stock board description, vendor implementation, and closest mainline driver.
3. **Test and verify:** A record of the single change, the tested build, and the observed result against the original hypothesis. Record what passed, what remains untested, and what would be needed to resume. Booting is separate from proving the feature works.

The agent moves to the next phase when its prerequisites are met. The operator confirms the target and authorizes device actions. In phase 3, record what would disprove the hypothesis before running the cheapest authorized test. A research-only request allows no device access, staging, next-boot selection, or reboot.

Widen the search after three refutations. A missing receipt does not refute the hypothesis. Verify reboot and runtime before retiring old deployments. Excluding userdata from a backup does not authorize erasing it.

The skill originated in one real port — mainline Linux on a Xiaomi Mi A3, running NixOS on kernels 7.1 and 7.2 — and now incorporates lessons from other ports while keeping each measurement's scope explicit.

## Examples

**Research prompt:** “Using the existing stock artifacts, research touchscreen support on this board. Don’t access or reboot the device.”

[`/port-research`](commands/port-research.md) returns a ranked plan with sources, the smallest test and what would disprove the hypothesis, plus a host-only gate report.

**Display bring-up:** On one port, the desktop still rendered after seven minutes, but the backlit panel stayed black. Preventing the display clocks from being parked and keeping the power domain on restored visible output. That verified [firmware framebuffer output](SKILL.md#3-implement-only-with-a-promising-evidence-backed-hypothesis), not the native panel driver.

**QEMU before a device boot:** One initramfs test used a missing root filesystem and a controlled interruption to exercise the log collector. It checked the collector's journal events and confirmed that both final snapshots succeeded. That tests the capture path, not the phone's hardware. Questions about real registers, wiring, or timing still need an authorized device test.

## Commands

Run a registered slash command or give your agent the linked file with `SKILL.md`. If a command calls another, load that file too. Without parallel subagents, use the same source ledger in order.

| Command                                                                 | When to use it                                          |
| ----------------------------------------------------------------------- | ------------------------------------------------------- |
| [`/port-setup`](commands/port-setup.md)                                 | Starting a port; stock evidence is still available      |
| [`/evidence-sweep`](commands/evidence-sweep.md)                         | A trial failed and you need evidence                    |
| [`/source-sweep`](commands/source-sweep.md)                             | You have evidence and need a hypothesis                 |
| [`/port-research`](commands/port-research.md)                           | You want to add an unproven capability                  |
| [`/flash-gate`](commands/flash-gate.md)                                 | A candidate is ready for its cheapest decisive test     |
| [`/port-loop`](commands/port-loop.md)                                   | Iterate within an agreed scope and refutation budget    |
| [`/port-upstream`](commands/port-upstream.md)                           | Prepare a patch or report for operator-owned submission |
| [`/port-cleanup`](commands/port-cleanup.md)                             | Reclaim host space without losing recovery inputs       |
| [`/port-update`](commands/port-update.md)                               | Upgrade and reverify port components                    |
| [`/linux-phone-porting-update`](commands/linux-phone-porting-update.md) | Update the installed skill, not the port                |

## Recommended folder structure

For a multi-device workspace, keep device-specific code out of the shared layers:

```text
kernel/<version>/     Upstream source and generic builder
soc/<vendor-soc>/     SoC-common patches and configuration
os/<platform>/        Device-neutral integration and tooling
devices/<model>/      Device assembly and state
  artifacts/          Gitignored backups, firmware harvests, and debug evidence
  logs/               One directory per device boot or deployment
```

Keep private backups, harvested firmware, and raw debug evidence under the device project's gitignored `artifacts/`. Record the origin, acquisition method, coverage, and hashes in an artifact manifest. Never publish private calibration or raw backups. Keep one log directory per boot or deployment so the evidence can be matched to the run.

### Adapting it to your port

Keep the device's identity, boot and recovery paths, target OS and session, and tested revisions in the device project's own records. Use the shared skill for the method: it tells the agent what to confirm and what evidence to collect. Keep device-specific patches, calibration, and state out of shared layers. The layout above is a reference, not a reason to move an existing workspace.

## Pairs well with

None are required. The skill falls back to source, documentation, and web search.

- `linux-kernel-development` or `linux-kernel-crash-debug` for kernel analysis.
- `find-docs` and `wigolo` for current docs and source sweeps.
- `graft` and `lei` for indexed source and Linux mailing-list archives.
- [Sashiko](https://github.com/sashiko-dev/sashiko) or [Boro](https://github.com/NVIDIA/boro) for optional patch review. Get approval for disclosure and spending; check findings against source or reproduction.

## Achievements

- **Qualcomm FastRPC:** [PR #393](https://github.com/qualcomm/fastrpc/pull/393) merged. It validates DSP-supplied buffer lengths.
- **Linux device tree:** The Qualcomm maintainer [applied the SM6125 `reserved_mem1` fix](https://lore.kernel.org/linux-arm-msm/179416929229.628393.16390580991068177022.b4-ty@b4/) to reserve another 28 MiB for firmware. The [patch records](https://github.com/torvalds/linux/commit/0733c22353a775b2b48e13a396ae8434cb17dc7a) about one lockup per 59 MiB of buffered reads before the change and no lockups across 691 GiB on the same workload afterwards.

## Changelog

- **2026-11-25**
  - **Workflow diagram.** Show the research-only exit and keep verification in phase 3.
  - **README and commands.** Use plainer language for setup, research, testing, and maintenance.
  - **Examples.** Show a research-only prompt and a measured display result.
  - **Achievements.** Link two contributions from porting work and state where each landed.
  - **Expanded guide.** Restore the workflow rationale, installation example, phase outputs, and advice for adapting the skill to a port. Add the QEMU and SM6125 test evidence.

Full history in [CHANGELOG.md](CHANGELOG.md).

## License

[CC BY 4.0](LICENSE) © [Roman Linev](https://rlinev.ru).
