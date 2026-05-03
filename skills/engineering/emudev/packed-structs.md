# Packed structs — hardware register layouts

Modeling hardware registers (LCDC, STAT, NR-series APU regs, PPUCTRL, etc.) with `packed struct` and `@bitCast` in Zig 0.16. Lean reference; not an exhaustive Zig packed-struct manual.

## The pattern

A hardware register is a fixed-width byte (or 16-bit word) where each bit (or bit range) has a named semantic. Model it as a `packed struct` with explicit bit-width fields, sized to match the underlying width.

Game Boy LCDC ($FF40):

```zig
// REF: pandocs/lcdc.html
pub const Lcdc = packed struct(u8) {
    bg_window_priority: bool, // bit 0: BG/window enable (DMG) / BG-OBJ priority (CGB)
    obj_enable:         bool, // bit 1
    obj_size:           ObjSize, // bit 2: 0 = 8x8, 1 = 8x16
    bg_tile_map:        TileMap, // bit 3: 0 = $9800-$9BFF, 1 = $9C00-$9FFF
    bg_window_tile_data: TileData, // bit 4
    window_enable:      bool, // bit 5
    window_tile_map:    TileMap, // bit 6
    lcd_enable:         bool, // bit 7

    pub const ObjSize = enum(u1) { eight_by_eight, eight_by_sixteen };
    pub const TileMap = enum(u1) { @"9800", @"9c00" };
    pub const TileData = enum(u1) { @"8800_signed", @"8000_unsigned" };
};

comptime {
    std.debug.assert(@sizeOf(Lcdc) == 1);
    std.debug.assert(@bitSizeOf(Lcdc) == 8);
}
```

Note `packed struct(u8)` — the explicit backing type. This makes `@bitCast` to/from `u8` zero-cost and well-defined.

## Reading and writing through the bus

The bus exposes register I/O as `u8` reads and writes. Convert at the boundary using `@bitCast`:

```zig
pub fn busRead(self: *Ppu, addr: u16) u8 {
    return switch (addr) {
        0xFF40 => @bitCast(self.lcdc),
        0xFF41 => @bitCast(self.stat),
        // ...
    };
}

pub fn busWrite(self: *Ppu, addr: u16, val: u8) void {
    switch (addr) {
        0xFF40 => self.writeLcdc(@bitCast(val)),
        0xFF41 => self.writeStat(@bitCast(val)),
        // ...
    }
}

fn writeLcdc(self: *Ppu, new: Lcdc) void {
    // QUIRK: turning LCD off mid-frame resets the LY counter and the
    // mode flags. The first frame after re-enable has different timing.
    // REF: pandocs/lcdc.html#bit-7-lcd-and-ppu-enable-bit
    if (self.lcdc.lcd_enable and !new.lcd_enable) {
        self.ly = 0;
        self.stat.mode = .hblank;
    }
    self.lcdc = new;
}
```

Centralizing the `@bitCast` at the bus boundary keeps the rest of the code reading typed fields. `self.lcdc.window_enable` is clearer than `(reg >> 5) & 1`.

## Bit-order gotchas

Zig packs `packed struct` fields starting from the least significant bit. **Always declare fields bit-0 first**, even when the hardware doc lists them bit-7 first.

If you copy a hardware doc table top-down, you'll get the order reversed. Verify with a `comptime` assertion:

```zig
comptime {
    // LCDC bit 7 is lcd_enable per pandocs.
    var x: Lcdc = @bitCast(@as(u8, 0x80));
    std.debug.assert(x.lcd_enable);
    std.debug.assert(!x.bg_window_priority);
}
```

Bake one of these per register so future-you can't misread the bit order silently.

## Read-only / write-only / mixed registers

Some hardware registers have asymmetric read/write semantics — e.g., NES PPUSTATUS ($2002) reads return status bits but writes are ignored; PPUDATA ($2007) reads have one-byte buffering except in the palette range.

Model the *storage* with a packed struct; encode the asymmetry in the read/write functions, **not** in the struct type. A register isn't "read-only at the type level" — it's read-only at the bus boundary, which is where the asymmetry is enforced.

```zig
pub fn busRead(self: *Ppu, addr: u16) u8 {
    return switch (addr) {
        0x2002 => blk: {
            // QUIRK: PPUSTATUS read clears bit 7 (vblank) and resets
            // the $2005/$2006 write toggle.
            // REF: nesdev/PPU_registers.html
            const out: u8 = @bitCast(self.status);
            self.status.vblank = false;
            self.write_toggle = false;
            break :blk out;
        },
        // ...
    };
}
```

## `@bitCast` alignment traps

`@bitCast` between equally-sized types is well-defined. Going through a pointer (`@ptrCast`) is **not** the same and carries alignment requirements. For register I/O, always `@bitCast` values, never `@ptrCast` register memory.

```zig
// GOOD:
const out: u8 = @bitCast(self.lcdc);

// BAD: pretends a u8 in a struct is a Lcdc; alignment-fragile,
// can violate strict aliasing.
const ptr: *Lcdc = @ptrCast(&self.lcdc_byte);
```

## Larger registers (16-bit, 24-bit)

Same pattern, larger backing type:

```zig
pub const Loopy = packed struct(u15) {
    coarse_x: u5,
    coarse_y: u5,
    nametable: u2,
    fine_y: u3,
};

comptime {
    std.debug.assert(@bitSizeOf(Loopy) == 15);
}
```

Note backing type matches the actual logical width. The NES "loopy V" register is 15 bits, not 16; declaring `packed struct(u15)` documents that and prevents accidental sign-extension when reading bit 15.

## OAM / sprite tables

Sprite entries are packed structs of 4 (Game Boy) or 4 (NES) or 4 (SNES) bytes. Declare an array of structs:

```zig
pub const Sprite = packed struct(u32) {
    y: u8,
    x: u8,
    tile: u8,
    attr: SpriteAttr,
};

pub const SpriteAttr = packed struct(u8) {
    palette_low: u3, // CGB: palette 0-7 / DMG: ignored
    bank: u1,         // CGB: VRAM bank
    palette_dmg: u1,  // DMG: 0 = OBP0, 1 = OBP1
    flip_x: bool,
    flip_y: bool,
    bg_priority: bool,
};

pub const Oam = struct {
    sprites: [40]Sprite, // Game Boy has 40 sprites
};
```

Indexing into the OAM byte-wise (which is how the bus exposes it) means doing `@bitCast` on a 32-bit slice of the array:

```zig
pub fn oamRead(self: *Ppu, addr: u8) u8 {
    const sprite_idx = addr / 4;
    const field_idx = addr % 4;
    const bytes: [4]u8 = @bitCast(self.oam.sprites[sprite_idx]);
    return bytes[field_idx];
}
```

Verify with `comptime` that the byte order matches the hardware spec — Game Boy OAM is `[Y, X, tile, attr]` in memory; declaring fields in that order makes `@bitCast` correct.

## Cross-references

- [comments.md](./comments.md) — every register definition cites pandocs / nesdev / fullsnes.
- [dispatch.md](./dispatch.md) — opcode handlers that read/write through the bus call into these typed accessors.
