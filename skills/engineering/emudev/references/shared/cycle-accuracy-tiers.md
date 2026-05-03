# Cycle accuracy tiers

Reference for decision #3 ("cycle accuracy tier"). Read before grilling on this decision; this file describes the candidates without prescribing one.

## The three tiers

### Instruction-stepped

Every CPU instruction completes atomically. The PPU/APU advance in bulk after each instruction completes, by the instruction's nominal cycle count.

- **Pros**: Simple to implement. Fast. CPU loop is a `while` over decode + execute. Acceptable for systems where the PPU is loosely coupled to the CPU (CHIP-8) or where the user does not care about pixel-precise rendering (web/mobile-style emulation of older systems).
- **Cons**: Cannot reproduce mid-instruction events. PPU bus contention with the CPU is invisible. DMC sample-stealing CPU stalls (NES) cannot be modeled. Sprite-0 hit (NES) timing approximations only. Mode-3 length variance (Game Boy) impossible.
- **Used by**: Older / faster commercial emulators in resource-constrained settings.

### M-cycle

The CPU is cycle-accurate at the **M-cycle** (machine cycle) granularity. On Game Boy, an M-cycle is 4 T-states. CPU operations decompose into 1-6 M-cycles each; bus reads/writes happen at specific M-cycles within an instruction. PPU/APU advance interleaved with CPU M-cycles.

- **Pros**: Sufficient for most Game Boy correctness — Mooneye and mealybug-tearoom suites mostly pass with M-cycle accuracy. Reasonable performance. Suitable middle ground.
- **Cons**: Sub-M-cycle PPU events (the GB pixel pipeline transitions inside an M-cycle) require T-state. Some mid-frame palette / OAM writes won't reproduce on M-cycle granularity.
- **Used by**: Sameboy at lower precision modes; most "good" Game Boy emulators ship M-cycle baseline.

### T-state (sub-M-cycle)

Every clock tick is modeled. CPU bus reads/writes happen at exact T-state offsets; PPU advances one dot per T-state; APU advances at sample-clock granularity within instructions.

- **Pros**: Reproduces every quirk the cycle-accuracy tier can in principle reproduce. Pixel-precise PPU (mode-3 length variance, mid-scanline writes), bus contention, DMC sample-stealing, etc.
- **Cons**: Slowest tier. CPU loop has more state than instruction- or M-cycle-stepped. Greenfield T-state implementations require significantly more bring-up work.
- **Used by**: SameBoy at full precision, bsnes/higan/ares for SNES, Mesen2 for NES.

## Picking a tier

The choice is rarely "as accurate as possible" — it's "what minimum tier passes the test-ROM set you commit to."

| System | M-cycle minimum baseline | T-state-required scenarios |
|---|---|---|
| Game Boy | Blargg, Mooneye acceptance, dmg-acid2 | mealybug-tearoom mode-3 timing, mid-scanline LCDC writes, sprite-priority edge cases |
| NES | nestest, Blargg sound suites | sprite-0 hit pixel-exact, DMC IRQ timing, MMC3 IRQ scanline-precise |
| SNES | Most title boots | Mode 7 mid-frame, HDMA tables, mid-scanline palette/OAM writes |

Recommended defaults:

- **Game Boy**: `m-cycle` if you commit to passing Mooneye but not mealybug. `t-state` if you want mealybug-tearoom + dmg-acid2 plus the harder mid-frame quirks.
- **NES**: `m-cycle` for boot-and-play of NROM/MMC1/MMC3 catalog. `t-state` for any ambition toward mid-scanline-effects games (Battletoads, Marble Madness, etc.).
- **SNES**: `t-state` is effectively the floor for any non-trivial title compatibility.

## Hard-to-upgrade-later

This is decision #3's "hard-to-reverse" justification. Going from instruction-stepped to M-cycle means rewriting the CPU loop and threading bus calls into mid-instruction. Going from M-cycle to T-state means doing it again, with finer granularity. Each upgrade is a multi-week refactor with broad ripple to PPU and APU.

Pick the highest tier you intend to ship.

## Mixed-tier considerations

It's tempting to pick "M-cycle CPU, T-state PPU." This often works for Game Boy — the PPU pixel pipeline runs on its own clock derived from the CPU clock; modeling the PPU at T-state granularity while keeping the CPU at M-cycle is coherent.

For NES this is harder — CPU and PPU share a 3:1 ratio that's tighter than Game Boy's bus contention. For SNES, the SPC700 audio CPU runs on an asynchronous clock that's notoriously hard to align without T-state precision.

If you decide on mixed tiers, document it in the ADR — the decision shape becomes "M-cycle CPU + T-state PPU + sample-rate APU" rather than a single tier choice.
