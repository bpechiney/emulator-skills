# NES references

Intentionally stubbed. Populate when NES emulator development begins (a separate repo from chippy / faux-boy).

Until then, the canonical references are:

- **nesdev wiki** — https://www.nesdev.org/wiki/
- **nesdev test ROMs** — https://github.com/christopherpow/nes-test-roms
- **Mesen2** source as citation prior art — https://github.com/SourMesen/Mesen2

When populating, mirror the Game Boy structure:

- `hardware-codenames.md` — 2C02 (NTSC) / 2C07 (PAL), front-loader / top-loader, RP2A03 / RP2A07, expansion-audio chip codenames (VRC6/7, MMC5, FDS, N163, S5B)
- `test-rom-catalog.md` — wiring for nestest, Blargg's NES suites, Mesen-test, Tom Harte 6502 JSON tests; pass-signal extraction patterns
- `nesdev-anchors.md` — curated list of nesdev wiki anchors (PPU registers, sprite-0 hit, DMC sample-stealing, etc.)
- `fidelity-scope-candidates.md` — NTSC / PAL / Famicom-audio scope, expansion-audio chip subset, light gun / Power Pad / Vs. System / FDS, peripheral support
