# Citation prefix templates

Shared reference for how citations are formatted. Read this when authoring a `REF` tag and you're unsure of the form.

## Goals

A citation should:

1. **Survive URL rot.** Pandocs / nesdev / fullsnes sometimes restructure. A bare URL that's broken in two years is a worse citation than no citation. Prefer verbatim-quote inclusion.
2. **Be greppable.** A future reader running `rg "REF: pandocs/oam_dma"` should find every related citation. Use stable prefixes (`pandocs/`, `nesdev/`, `fullsnes/`, `gbdev/`, `mooneye/`, `mealybug/`, `bsnes/`, `ares/`, `sameboy/`, etc.).
3. **Be locatable.** When the URL still works, the reader can click through. Include the URL as a fragment of the citation (in the comment block, not the prefix).

## The Mesen-style pattern (preferred)

Mesen2 cites the nesdev wiki by pasting verbatim sentences as `// "..."` blocks alongside the prefix. This is robust to URL rot because the cited text is in the source.

```zig
// REF: pandocs/oam_dma.html
// "DMA copies 160 bytes (40 OAM entries × 4 bytes each) from source
//  to OAM. The source can be any 0x100-aligned address. The CPU is
//  effectively halted (only HRAM access works) for 160 cycles."
fn startOamDma(...) { ... }
```

The reader sees the rule without hitting the network. Two years from now the URL might 404, but the comment still tells you what the rule was.

## Prefix conventions

| Prefix | Source |
|---|---|
| `pandocs/` | https://gbdev.io/pandocs/ — Game Boy Pandocs |
| `nesdev/` | https://www.nesdev.org/wiki/ |
| `fullsnes/` | https://problemkaputt.de/fullsnes.htm — SNES Hardware Reference |
| `gbdev/` | https://gbdev.io/ — Game Boy development community broadly |
| `gbdev-codenames/` | https://gbdev.io/pandocs/Specifications.html — for the DMG/MGB/SGB/SGB2/CGB/AGB canon |
| `mooneye/` | https://github.com/Gekkio/mooneye-test-suite — Mooneye test ROM |
| `mealybug/` | https://github.com/mattcurrie/mealybug-tearoom-tests |
| `blargg/` | https://github.com/retrio/gb-test-roms — Blargg suite |
| `nestest/` | https://github.com/christopherpow/nes-test-roms |
| `bsnes/` | bsnes / bsnes-plus source (architecture references, prior art) |
| `ares/` | ares emulator source (architecture references) |
| `sameboy/` | https://github.com/LIJI32/SameBoy — frequently the gold standard for GB |
| `mesen2/` | https://github.com/SourMesen/Mesen2 — citation-style prior art |
| `snes9x/` | Snes9x source (often cited as anti-pattern) |
| `adr/` or `docs/adr/` | This repo's ADR directory |

## Stable anchors

When citing pandocs / nesdev / fullsnes, prefer **section anchors** over plain page URLs. Pandocs uses lower-snake-case anchors (`#oam-dma`, `#interrupts-and-frame-counter`). The anchor names are more stable than full URLs across structure changes.

For Game Boy specifically, the curated list of useful pandocs anchors lives in `references/gameboy/pandocs-anchors.md`. That file is treated as living — when an anchor changes, update it there, then references throughout the codebase still work.

## ADR citations

For internal architectural decisions, cite the ADR file path (not a short ID). ADR file paths are stable; numeric IDs in shorthand are less greppable.

```zig
// REF: docs/adr/0007-per-milestone-prd.md
// Per-milestone PRDs are the unit of planning.
```

## Don't cite

- The Zig stdlib documentation. The agent has stdlib knowledge; citations there add noise.
- "Common knowledge" CPU concepts (what `ADD A, B` does in Z80 / SM83). Cite only when behavior is non-obvious or the doc disambiguates.
- Your own code in another file ("see foo.zig"). Use `///` doc comments and module imports instead.
