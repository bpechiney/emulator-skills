# Game Boy test-ROM catalog (wiring focus)

This file is about **wiring**: where each suite's submodule lives, how to extract its pass/fail signal, and how to integrate it into `zig build test`. It is not an enumeration of test names — those are well-documented at the upstream repos.

## The suites

### Blargg

- **Upstream**: https://github.com/retrio/gb-test-roms
- **Submodule path**: `tests/test-roms/blargg/`
- **Suites**: `cpu_instrs`, `instr_timing`, `mem_timing`, `oam_bug`, `dmg_sound`, `cgb_sound`, `interrupt_time`, `halt_bug`
- **Pass signal**: ROM prints suite results to the **Game Boy serial port**. The harness instantiates a serial-capture device, runs to a max-cycles budget, scans the captured bytes for `Passed`. If the buffer contains `Failed`, fail the test with the buffer contents printed.
- **Max cycles**: ~400M cycles (~ 100s of game-time) is generous; tighten per-suite as the implementation matures.
- **Order to attempt**: `cpu_instrs` first (smallest correctness baseline), then `instr_timing`, then memory and sound.

### Mooneye Test Suite

- **Upstream**: https://github.com/Gekkio/mooneye-test-suite
- **Submodule path**: `tests/test-roms/mooneye/`
- **Pass signal**: After the test ROM completes, the CPU registers contain a magic pattern. Pass = `B = 3, C = 5, D = 8, E = 13, H = 21, L = 34` (the Fibonacci sequence). Fail = different pattern. The harness asserts on register state at the suite's specified cycle count.
- **Subsuites**: `acceptance/`, `emulator-only/`, `manual-only/`, `misc/`. Start with `acceptance/`.
- **Notes**: Mooneye is the gold-standard *correctness* suite; assumes M-cycle accuracy minimum. Some tests in `manual-only/` require human verification — exclude from CI.

### Mealybug Tearoom

- **Upstream**: https://github.com/mattcurrie/mealybug-tearoom-tests
- **Submodule path**: `tests/test-roms/mealybug/`
- **Pass signal**: Framebuffer hash after running the test for its specified cycle count. Each test has a known-correct expected PNG / hash. The harness captures the framebuffer and compares.
- **Notes**: Mealybug pushes mid-frame timing. Many tests require T-state accuracy. Skip in M-cycle-only builds; assert in T-state builds.

### dmg-acid2

- **Upstream**: https://github.com/mattcurrie/dmg-acid2
- **Submodule path**: `tests/test-roms/dmg-acid2/`
- **Pass signal**: Single-frame framebuffer hash. Acid-test pattern; if your PPU is correct enough, the rendered face matches the reference.
- **Notes**: M-cycle accuracy is enough for most of dmg-acid2; some sub-pixel details require T-state.

### Tom Harte JSON tests

- **Upstream**: https://github.com/SingleStepTests/sm83 (Game Boy SM83)
- **Submodule path**: `tests/test-roms/sm83-json/`
- **Pass signal**: Each JSON file is `{ initial: { cpu state, ram }, final: { cpu state, ram }, cycles: [...] }`. The harness loads initial state, runs ONE instruction, asserts final state matches.
- **Notes**: Per-opcode coverage — the test set is large (~2.5GB unzipped). Useful as a CPU bring-up gate rather than a routine CI run; tag this suite as `--slow` in build flags and run on demand.

## Suggested wiring shape

```zig
// In build.zig, after building the core module:

const test_rom_runner = b.addExecutable(.{
    .name = "test-rom-runner",
    .root_module = b.createModule(.{
        .root_source_file = b.path("tests/test_rom_runner.zig"),
        .target = target,
        .optimize = optimize,
    }),
});
test_rom_runner.root_module.addImport("core", core_mod);

const test_step = b.step("test", "Run all tests");

inline for (.{ "blargg", "mooneye", "mealybug", "dmg-acid2" }) |suite| {
    const run = b.addRunArtifact(test_rom_runner);
    run.addArg("--suite");
    run.addArg(suite);
    run.addArg("--timeout-cycles");
    run.addArg("400000000");
    run.has_side_effects = true; // re-run every time
    test_step.dependOn(&run.step);
}

const sm83_step = b.step("test-sm83", "Run Tom Harte SM83 JSON tests (slow)");
const sm83_run = b.addRunArtifact(test_rom_runner);
sm83_run.addArg("--suite");
sm83_run.addArg("sm83-json");
sm83_step.dependOn(&sm83_run.step);
```

`zig build test` runs the four fast suites; `zig build test-sm83` runs the slow per-opcode bring-up suite on demand.

## Test-runner anatomy

`tests/test_rom_runner.zig` is a small Zig program that:

1. Parses `--suite`, `--timeout-cycles`, `--rom` (optional, otherwise iterates all ROMs in the suite directory).
2. For each ROM:
   - Constructs a fresh `Console` with a serial capture (or framebuffer capture, or register-snapshot, depending on suite).
   - Runs up to the cycle budget.
   - Extracts the pass signal via the suite-appropriate method.
   - Records pass/fail.
3. Exits with status 0 if all pass, non-zero with a per-ROM summary if any fail.

Keep the runner small. Suite-specific logic lives in `runners/{blargg,mooneye,mealybug,acid,sm83}.zig` modules.

## Initial baseline

For a greenfield Game Boy emulator (faux-boy):

- **M0 baseline**: `blargg-cpu_instrs-01-special` passes. (Smallest meaningful correctness signal.)
- **M1 baseline**: All `blargg-cpu_instrs-*` pass. CPU is "done."
- **M2 baseline**: `blargg-instr_timing` passes. Cycle-counts are correct.
- **M3 baseline**: All Mooneye `acceptance/*` pass.
- **M4 baseline**: `dmg-acid2` passes.
- **M5 baseline**: Mealybug PPU subset passes (requires T-state if not already on it).

Land each baseline in its own `acceptance` test that asserts the milestone hasn't regressed.
