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

When slicing mapper work for `/to-issues`, slice **register-by-register**, not mapper-by-mapper. One slice per mapper register is the right granularity; "implement MBC1" is too coarse.

## `comptime Bus: type` for CPU↔Bus

The CPU is parameterized over the bus type at compile time. There's typically only one concrete bus per build, so monomorphization wins outright.

```zig
pub fn Cpu(comptime Bus: type) type {
    return struct {
        const Self = @This();

        pc: u16,
        sp: u16,
        a: u8,
        bc: u16,
        de: u16,
        hl: u16,
        flags: Flags,

        // Half-register accessors (cpu.b(), cpu.c(), cpu.h(), cpu.l(), ...)
        // elided. Pair storage matches the SM83 instructions that act on BC,
        // DE, HL as 16-bit operands; halves are computed via @truncate /
        // @as(u16, x) << 8.

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

## Cross-references

- [dispatch.md](./dispatch.md) — opcode dispatch shape that calls into the bus type chosen here.
- [packed-structs.md](./packed-structs.md) — most polymorphic register reads/writes flow through the mapper or bus boundary.
- [testing.md](./testing.md) — test discipline this skill defers to (test ROMs, determinism, save-state round-trip).
