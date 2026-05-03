# Dispatch

CPU opcode dispatch in cycle-accurate emulators in Zig 0.16.

## The recommended pattern: labeled `switch` with `continue :state`

Zig 0.16's labeled `switch` form is the recommended dispatch pattern for opcode loops. Zig's own tokenizer measured a +13% throughput gain converting from a giant `switch` to this form. Of ~12 surveyed Zig-language emulators in late 2025, **zero** used it — the pattern is non-default. Greenfield emulator projects should adopt it from day one.

```zig
const Op = enum(u8) {
    nop      = 0x00,
    ld_bc_d16 = 0x01,
    ld_bc_a   = 0x02,
    inc_bc    = 0x03,
    // ... (full Game Boy opcode table elided)
};

pub fn step(cpu: *Cpu, bus: anytype) void {
    var op: Op = @enumFromInt(bus.read(cpu.pc));
    cpu.pc +%= 1;

    dispatch: switch (op) {
        .nop => {
            cpu.tick(4);
            return;
        },
        .ld_bc_d16 => {
            const lo = bus.read(cpu.pc); cpu.pc +%= 1;
            const hi = bus.read(cpu.pc); cpu.pc +%= 1;
            cpu.bc = (@as(u16, hi) << 8) | lo;
            cpu.tick(12);
            return;
        },
        .ld_bc_a => {
            bus.write(cpu.bc, cpu.a);
            cpu.tick(8);
            return;
        },
        .inc_bc => {
            cpu.bc +%= 1;
            cpu.tick(8);
            // Tail-chain into the next instruction without leaving
            // the dispatch frame.
            op = @enumFromInt(bus.read(cpu.pc));
            cpu.pc +%= 1;
            continue :dispatch op;
        },
        // ...
    }
}
```

Key points:

- `dispatch:` labels the `switch`. `continue :dispatch op;` re-enters with a new tag value without unwinding the frame. The compiler can specialize the indirect jump per-arm.
- The labeled `continue` is what gives the compiler the leverage. A bare `switch` reached the +13% only after the labeled form landed.
- Tail-chaining (`continue :dispatch`) is most useful for opcodes that naturally flow into the next instruction without a fetch boundary (e.g., decoded prefix instructions that reuse the dispatch frame).

## When to use a function-pointer table instead

Function-pointer tables compete with labeled `switch` on:

| Axis | Labeled `switch` | Function-pointer table |
|---|---|---|
| Per-opcode code locality | High (all arms in one fn) | Low (each opcode is a separate fn) |
| Compiler specialization | High (visible at optimization) | Lower (function calls obscure flow) |
| Build-time | Single fn compiles slower | Many small fns compile in parallel |
| Profiling granularity | One symbol | One symbol per opcode |
| Runtime patching | No | Yes (swap entries at runtime) |

**Recommendation**: labeled `switch` for the base opcode table. Function-pointer tables only when you need runtime patching (e.g., debugger trap insertion replacing one opcode with a breakpoint stub) — and even then prefer a `comptime` parameter that selects between a "trapping" and "non-trapping" dispatch loop, instantiated as separate concrete functions.

## CB-prefix opcodes

Game Boy and similar CPUs have a 256-opcode prefix table reached via a single byte (0xCB on Game Boy). Two reasonable shapes:

**Shape 1 — nested labeled `switch`:**

```zig
.cb_prefix => {
    const sub: CbOp = @enumFromInt(bus.read(cpu.pc));
    cpu.pc +%= 1;
    switch (sub) {
        .rlc_b => { ... cpu.tick(8); return; },
        .rlc_c => { ... cpu.tick(8); return; },
        // ...
    }
},
```

**Shape 2 — separate function:**

```zig
.cb_prefix => {
    return cb.step(cpu, bus);
},
```

Shape 2 is simpler; Shape 1 keeps the locality argument. For greenfield Game Boy work, **prefer Shape 2** — the CB sub-table is large enough that inlining it inflates the dispatch fn for marginal gain. Reserve nested labeled `switch` for cases where the prefix sub-table is small (single-digit entries).

## `@branchHint` for hot/cold paths

Zig 0.16 supports `@branchHint(.likely)` / `@branchHint(.unlikely)` / `@branchHint(.cold)`. Use sparingly — the compiler's profile-guided heuristics are usually better than hand-tuning. Reserve hints for:

- The `.cold` arm of an interrupt-check branch (interrupts are rare relative to instructions).
- Error/trap arms in CPU dispatch (`@panic`, `unreachable`).

```zig
if (cpu.pending_irq) {
    @branchHint(.unlikely);
    return cpu.serviceIrq(bus);
}
```

## Pool patterns for hot-path objects

The hot path does not allocate (standing rule #4). Objects with fixed cardinality (e.g., 40 OAM sprites, 4 APU channels, N mapper-bank windows) are pre-allocated as array fields on the console struct and indexed by `usize`. No `ArrayList`, no `std.heap`. When you need a "freelist" for transient work (e.g., a queue of pending DMA bytes), pre-size it at boot and overflow into a panic — overflow is a bug, not a fallback.

```zig
pub const Apu = struct {
    square1: SquareChannel,
    square2: SquareChannel,
    wave: WaveChannel,
    noise: NoiseChannel,
    // No .channels: []Channel — fixed cardinality, named fields are clearer.
};
```

## Cross-references

- [comments.md](./comments.md) — every `case` arm whose timing is non-obvious carries a `REF` or `QUIRK` tag.
- [polymorphism.md](./polymorphism.md) — for the bus type the dispatch operates on.
- [packed-structs.md](./packed-structs.md) — for register reads/writes inside opcode bodies.
