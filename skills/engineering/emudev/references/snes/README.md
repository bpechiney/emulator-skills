# SNES references

Intentionally stubbed. Populate when SNES emulator development begins (a separate repo from chippy / faux-boy / nes-emulator).

Until then, the canonical references are:

- **fullsnes** by Martin Korth — https://problemkaputt.de/fullsnes.htm
- **bsnes / higan / ares** source as citation prior art — https://github.com/higan-emu/ares
- **PeterLemon SNES** test suite — https://github.com/PeterLemon/SNES

When populating, mirror the Game Boy structure:

- `hardware-codenames.md` — 1-CHIP / 2-CHIP / 1-CHIP-3-CHIP main board revisions, pre/post-1991 PPU revisions, DSP-1 / DSP-1A / DSP-1B coprocessor revisions, GSU-1 / GSU-2 (SuperFX), SA-1, S-DD1, ST010 / ST011 / ST018, MSU-1, S-RTC
- `test-rom-catalog.md` — wiring for PeterLemon, Tom Harte 65816 JSON tests; pass-signal extraction; the PPU / DMA / HDMA / Mode 7 / SPC700 sub-suites
- `fullsnes-anchors.md` — curated list of fullsnes section anchors (DMA / HDMA / Mode 7 / VRAM access / SPC700 / DSP audio)
- `fidelity-scope-candidates.md` — main-board revision scope, coprocessor scope (DSP / SuperFX / SA-1 / S-DD1 / ST010 / MSU-1), peripheral scope (Multitap, Mouse, Super Scope, Justifier), SPC700 sync precision (cycle-exact vs catch-up), audio analog characteristics
