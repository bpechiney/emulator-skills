# Game Boy test-ROM catalog

Mechanical reference for the Game Boy test-ROM suites — upstream URLs, pass-signal extraction conventions, and cycle-accuracy tier implications. The model otherwise has to look these up.

What this file does **not** cover: the actual `build.zig` wiring (see [build.md](../../build.md)), the submodule path on disk (whatever you set in `docs/agents/emudev.md` as `test_rom_root`), and your bring-up roadmap (your plan-of-record, not this file's call).

## Blargg

- **Upstream**: https://github.com/retrio/gb-test-roms
- **Pass signal**: Game Boy serial port. Instantiate a serial-capture device and scan the captured bytes for `Passed` / `Failed` after a per-suite cycle budget.
- **Subsuites you'll see**: `cpu_instrs`, `instr_timing`, `mem_timing`, `oam_bug`, `dmg_sound`, `cgb_sound`, `interrupt_time`, `halt_bug`.

## Mooneye Test Suite

- **Upstream**: https://github.com/Gekkio/mooneye-test-suite
- **Pass signal**: Magic register pattern at completion. Pass = `B = 3, C = 5, D = 8, E = 13, H = 21, L = 34` (the Fibonacci sequence). Anything else is a fail.
- **Subsuites**: `acceptance/`, `emulator-only/`, `manual-only/`, `misc/`. `manual-only/` requires human verification — exclude from CI.
- **Tier requirement**: M-cycle accuracy minimum.

## Mealybug Tearoom

- **Upstream**: https://github.com/mattcurrie/mealybug-tearoom-tests
- **Pass signal**: Framebuffer hash at the suite's specified cycle count, compared against a vendored expected PNG / hash.
- **Tier requirement**: Most tests need T-state accuracy. Skip on M-cycle-only builds.

## dmg-acid2

- **Upstream**: https://github.com/mattcurrie/dmg-acid2
- **Pass signal**: Single-frame framebuffer hash. The acid-test face pattern.
- **Tier requirement**: M-cycle is enough for most of it; sub-pixel details require T-state.

## Tom Harte SM83 JSON tests

- **Upstream**: https://github.com/SingleStepTests/sm83
- **Pass signal**: Per-instruction. Each JSON file is `{ initial, final, cycles }`; load the initial state, run one instruction, assert the final state matches.
- **Notes**: Useful as a CPU bring-up gate, not routine CI — the suite is large (~2.5GB unzipped). Tag separately in `build.zig` and run on demand.

## Vendoring

Vendor as **git submodules** under the `test_rom_root` declared in `docs/agents/emudev.md`. Submodules pin to specific upstream commits — you control when test-ROM expectations change. Don't vendor as tarballs.

Most test ROMs are permissively licensed but require attribution; the submodule pattern preserves the upstream LICENSE.
