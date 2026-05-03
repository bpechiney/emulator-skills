# Game Boy hardware codenames

The gbdev community has a canonical codename for each Game Boy hardware revision. **Use the codename, not the marketing name** — `CGB`, not "GBC"; `MGB`, not "Game Boy Pocket".

Greppability matters: `rg "HW\[CGB" src/` finds every CGB-specific code path. If half your code says `HW[CGB]` and half says `HW[GBC]`, the search misses things.

## The canon

| Codename | Marketing name | Year | Notes |
|---|---|---|---|
| **DMG** | Game Boy | 1989 | Original, monochrome, 4 MHz CPU. Has the OAM bug. |
| **MGB** | Game Boy Pocket | 1996 | Same SoC family as DMG; some minor analog audio differences. Same CPU, no OAM bug fix. |
| **SGB** | Super Game Boy | 1994 | DMG-on-SNES adapter. CPU clocks at SNES sub-clock (slightly higher), supports the SGB packet protocol, custom border feature. Audio routed through SNES. |
| **SGB2** | Super Game Boy 2 | 1998 | Like SGB but with a separate clock crystal — CPU runs at proper DMG speed (most SGB games race because of the clock skew, SGB2 fixes this). |
| **CGB** | Game Boy Color | 1998 | New SoC. Backwards compatible with DMG cartridges. KEY1 register doubles CPU clock. CGB-DMA, palette compatibility, RAM banks. OAM bug removed. |
| **AGB** | Game Boy Advance (in GBC mode) | 2001 | When booted with a GBC cartridge, behaves as CGB-equivalent with minor analog audio differences. Strictly: AGB-CGB mode. |

For SGB-specific code: distinguish `SGB` (1994, clock-skewed) from `SGB2` (1998, accurate). Some games rely on the SGB clock skew; emulating SGB2 cleanly is generally easier and more correct.

## Common groupings

Group codenames in the bracketed `HW[]` tag when a behavior applies to multiple revisions:

```zig
// HW[DMG,MGB]: OAM bug — increment/decrement of HRAM-adjacent OAM
// during sprite scan corrupts OAM. Removed on CGB.

// HW[SGB,SGB2]: SGB packet protocol via JOYPAD reads. Used by some
// games for border / palette commands.

// HW[CGB,AGB]: KEY1 register controls CPU clock doubling.

// HW[!CGB,!AGB]: behaviors that apply to original-Game-Boy family
// (DMG, MGB, SGB, SGB2) but not the color hardware.
```

## ROM compatibility byte

The Game Boy cartridge header at `$0143` indicates CGB-compatibility:

- `$80` — CGB-aware (works on DMG too)
- `$C0` — CGB-only (won't run on DMG)
- Anything else (or absent on older carts) — DMG game

Decision #6 (fidelity scope) governs whether your emulator respects this byte, falls back to DMG when the byte is missing, refuses to run CGB-only ROMs in a DMG-only build, etc.

## SoC vs hardware revision

Internally the Game Boy SoC has minor manufacturing revisions even within a single codename — `CPU_0` vs `CPU_A` vs `CPU_B`, etc. Most emulators ignore these (they affect only sub-microsecond analog timing). If you encounter a hardware quirk that varies *within* a codename, document it as `HW[DMG_CPU_B]` or similar — but expect this to be rare.

## REF

- gbdev/Pandocs § Specifications: hardware codename canon
- pandocs/specs.html
