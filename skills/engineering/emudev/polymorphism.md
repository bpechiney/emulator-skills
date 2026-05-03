# Polymorphism

Choosing between tagged union, vtable, and `comptime` monomorphization for the recurring polymorphism boundaries in an emulator.

## The decision matrix

| Boundary | Cardinality | Open or closed? | Recommended shape |
|---|---|---|---|
| **Mappers / cartridges** | NES ~250, Game Boy ~10, SNES ~30 | Closed historical set — no new ones will be invented | Tagged `union(enum)` with `inline else` dispatch |
| **CPU↔Bus** | 1 per concrete bus type (typically just *the* bus) | Closed at compile time within a build | `comptime Bus: type` (monomorphized) |
| **Frontend** (renderer, audio, input) | Many possible (raylib, SDL, TUI, headless, web) | Open — users may add new ones | Vtable (function-pointer struct) |
| **Multi-system Console** | If you ever build one supervisor that can host both GB and NES cores | Closed but small | Tagged `union(enum)` |

The temptation to use a vtable everywhere because "polymorphism is polymorphism" leaves performance on the table for the hot-path closed-set cases (mappers, bus). The temptation to use tagged unions everywhere creates impossible-to-extend frontends. **Match the shape to the openness of the set.**

## Tagged `union(enum)` with `inline else`

For mappers, the tagged union with `inline else` lets the compiler specialize the dispatch per active mapper while preserving the closed-set guarantee at the type system level.

```zig
pub const Mapper = union(enum) {
    rom_only: RomOnly,
    mbc1: Mbc1,
    mbc3: Mbc3,
    mbc5: Mbc5,
    huc1: Huc1,
    // ...

    pub fn read(self: *Mapper, addr: u16) u8 {
        switch (self.*) {
            inline else => |*m| return m.read(addr),
        }
    }

    pub fn write(self: *Mapper, addr: u16, val: u8) void {
        switch (self.*) {
            inline else => |*m| m.write(addr, val),
        }
    }
};
```

`inline else` instantiates one specialized arm per active variant. The compiler can inline the call into each arm — the resulting code is comparable to a bare function call into `Mbc3.read` once the runtime tag is known.

### Slicing mapper work

When implementing a new mapper, **slice register-by-register, not mapper-by-mapper**. Each mapper has 2-6 register windows; one slice per register is the right granularity. A "implement MBC1" issue is too coarse — break it into "MBC1 ROM bank lower 5 bits", "MBC1 RAM/ROM mode", "MBC1 upper 2 bits with mode-1 bank-0 alias", etc. Each slice has a discrete test ROM that exercises it.

## `comptime Bus: type` for CPU↔Bus

The CPU is parameterized over the bus type at compile time. There's typically only one concrete bus per build, so monomorphization wins outright.

```zig
pub fn Cpu(comptime Bus: type) type {
    return struct {
        const Self = @This();

        pc: u16,
        sp: u16,
        a: u8, b: u8, c: u8, d: u8, e: u8, h: u8, l: u8,
        flags: Flags,

        pub fn step(self: *Self, bus: *Bus) void {
            const op = bus.read(self.pc);
            // ... dispatch ...
        }
    };
}

// Usage:
const Console = struct {
    bus: Bus,
    cpu: Cpu(Bus),
};
```

Trade-off: testability. With `comptime Bus`, mocking the bus means defining a `MockBus` type with the same surface. That's slightly more friction than passing a vtable, but the duck-typing structural check Zig performs is enough for tests — `MockBus` doesn't need a formal interface declaration.

If you decide testability outweighs the perf gain, use a vtable. But for cycle-accurate work, the bus is on the hottest of paths and should be monomorphized.

## Vtable (function-pointer struct) for frontends

Frontends are open: users may swap raylib for SDL, add a TUI, add a headless trace dumper, add a web-assembly bridge. Vtable is the right shape.

```zig
pub const Renderer = struct {
    ctx: *anyopaque,
    vtable: *const Vtable,

    pub const Vtable = struct {
        present: *const fn (ctx: *anyopaque, framebuffer: []const u32) void,
        deinit: *const fn (ctx: *anyopaque) void,
    };

    pub fn present(self: Renderer, framebuffer: []const u32) void {
        self.vtable.present(self.ctx, framebuffer);
    }

    pub fn deinit(self: Renderer) void {
        self.vtable.deinit(self.ctx);
    }
};

pub const RaylibRenderer = struct {
    // ... raylib-specific state ...

    pub fn renderer(self: *RaylibRenderer) Renderer {
        return .{ .ctx = self, .vtable = &vtable };
    }

    const vtable: Renderer.Vtable = .{
        .present = present,
        .deinit = deinit,
    };

    fn present(ctx: *anyopaque, fb: []const u32) void {
        const self: *RaylibRenderer = @ptrCast(@alignCast(ctx));
        // ... draw ...
    }
    fn deinit(ctx: *anyopaque) void {
        const self: *RaylibRenderer = @ptrCast(@alignCast(ctx));
        // ... clean up ...
    }
};
```

Vtables aren't on the hot path (per-frame `present` calls are cheap), so the indirection is fine.

## Hybrid: when does it make sense?

For the **multi-system Console** case (a supervisor that hosts both Game Boy and NES cores in the same process), tagged `union(enum)` with two arms is fine. If you later want to load arbitrary cores at runtime (a plugin model), introduce a vtable. Until then, keep it closed.

For mapper coprocessors (DSP-1, SuperFX, SA-1 on SNES), the tagged-union pattern still works — they're a closed set. You don't need a vtable just because the cardinality crept up.

## Revision gating reuse

Decision #6 (fidelity scope) often ends up reusing the tagged-union pattern. If your Game Boy emulator supports DMG and CGB:

```zig
pub const Revision = union(enum) {
    dmg: Dmg,
    cgb: Cgb,

    pub fn step(self: *Revision, ...) void {
        switch (self.*) {
            inline else => |*r| r.step(...),
        }
    }
};
```

Or, if revision is fixed at compile time per build (e.g., a feature flag selects DMG-only or CGB build), use `comptime Revision` instead. The choice between runtime (`union(enum)`) and compile-time (`comptime`) gating is itself part of decision #6.

## Cross-references

- [dispatch.md](./dispatch.md) — opcode dispatch shape that calls into the bus type chosen here.
- [packed-structs.md](./packed-structs.md) — most polymorphic register reads/writes flow through the mapper or bus boundary.
- [testing.md](./testing.md) — `MockBus` patterns for unit-testing the CPU when the production bus is a `comptime` parameter.
