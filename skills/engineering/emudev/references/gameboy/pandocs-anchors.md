# Pandocs anchors (living)

Curated list of pandocs section anchors used in citations. Treat this as living — when pandocs restructures, update entries here once, and existing `REF: pandocs/...` citations still resolve.

Pandocs root: https://gbdev.io/pandocs/

## CPU

| Anchor | Subject |
|---|---|
| `pandocs/cpu_instruction_set.html` | Full instruction set reference |
| `pandocs/cpu_registers_and_flags.html` | A/F/B/C/D/E/H/L/SP/PC, flag layout |
| `pandocs/halt.html` | HALT instruction + the HALT-bug |
| `pandocs/stop.html` | STOP instruction (2-byte width, edge cases) |
| `pandocs/interrupts.html` | IF/IE registers, interrupt servicing, IME |
| `pandocs/power_up_sequence.html` | Boot ROM behavior, post-boot register state |

## Memory map and bus

| Anchor | Subject |
|---|---|
| `pandocs/memory_map.html` | Full address space layout |
| `pandocs/io_registers.html` | I/O register summary at $FF00-$FF7F |
| `pandocs/oam_dma.html` | DMA transfer to OAM (160 cycles, HRAM-only CPU) |
| `pandocs/cgb_dma.html` | CGB-specific HDMA / general-purpose DMA |
| `pandocs/echo_ram.html` | Echo RAM mirror at $E000-$FDFF |

## PPU

| Anchor | Subject |
|---|---|
| `pandocs/lcdc.html` | LCDC ($FF40) bit-by-bit reference |
| `pandocs/lcd_status.html` | STAT ($FF41), modes 0-3 |
| `pandocs/lcd_position_and_scrolling.html` | LY / LYC / SCY / SCX / WX / WY |
| `pandocs/pixel_fifo.html` | Pixel pipeline / mode-3 length variance |
| `pandocs/oam.html` | OAM ($FE00-$FE9F), sprite attributes |
| `pandocs/palettes.html` | DMG palette regs / CGB palette RAM |
| `pandocs/rendering.html` | Per-scanline rendering walk-through |
| `pandocs/vram_tile_data.html` | Tile data layout in VRAM |
| `pandocs/vram_tile_maps.html` | BG/window tile map layout |

## APU

| Anchor | Subject |
|---|---|
| `pandocs/audio_overview.html` | Channels 1-4 overview |
| `pandocs/audio_registers.html` | NR-series register reference |
| `pandocs/audio_details.html` | Sweep, length, envelope details + edge cases |

## Cartridges and mappers

| Anchor | Subject |
|---|---|
| `pandocs/the_cartridge_header.html` | Cartridge header layout |
| `pandocs/mbcs.html` | MBC overview |
| `pandocs/nombc.html` | No-MBC (ROM-only) |
| `pandocs/mbc1.html` | MBC1 — ROM/RAM banking + mode-1 quirks |
| `pandocs/mbc2.html` | MBC2 |
| `pandocs/mbc3.html` | MBC3 — RTC |
| `pandocs/mbc5.html` | MBC5 — 8MB ROM, 128KB RAM |
| `pandocs/mbc7.html` | MBC7 — accelerometer (Kirby Tilt 'n' Tumble) |
| `pandocs/huc1.html` | HuC1 |
| `pandocs/huc3.html` | HuC3 |

## CGB-specific

| Anchor | Subject |
|---|---|
| `pandocs/cgb_speed_switch.html` | KEY1, double-speed mode |
| `pandocs/vram_banking.html` | CGB VRAM bank switching |
| `pandocs/wram_banking.html` | CGB work RAM bank switching |
| `pandocs/cgb_palettes.html` | CGB palette RAM ($FF68-$FF6B) |
| `pandocs/object_priority_and_conflict.html` | CGB sprite priority |

## Specifications / canon

| Anchor | Subject |
|---|---|
| `pandocs/specs.html` | Hardware codename canon (DMG/MGB/SGB/SGB2/CGB/AGB) |

## Maintenance

When pandocs renames an anchor:

1. Update this file with the new path.
2. Run `rg "pandocs/<old-anchor>" src/` and update inline citations.
3. Commit as a single "refresh pandocs anchors" commit so the diff is reviewable.

When you find a useful anchor not in this list, add it. The file's job is to be the locator-of-record.
