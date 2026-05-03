# Game Boy fidelity-scope candidates

Reference for decision #6 ("fidelity scope") applied to Game Boy. This file enumerates the candidate scope choices; **it does not prescribe one** — that's `/grill-with-docs`'s job, per repo.

## Scope axes

The fidelity-scope decision has multiple sub-axes. For each, decide what's in scope and what's deliberately deferred.

### Hardware revisions

| Scope choice | What's in / what's out | Implications |
|---|---|---|
| **DMG-only** | Just original Game Boy. No CGB code paths. | Smallest scope. Many CGB-only games refuse to run. CGB cartridge-header byte ($0143 = $80 / $C0) is read-only. Save-state schema simpler. |
| **DMG + CGB** | Both, with revision selected per ROM (or compile-time flag). | The default for "good" GB emulators. Save-state schema must encode revision. KEY1 / VRAM banks / palette RAM / HDMA / object-priority all gated. |
| **All six** (DMG/MGB/SGB/SGB2/CGB/AGB) | Full canon. | Adds SGB packet protocol, MGB / AGB analog audio differences, SGB clock skew, SGB2 fixed clock. Highest scope; most accurate analog audio is part of the win. |

Recommended default for a greenfield project: **DMG + CGB**. Adding MGB / AGB later is mostly a matter of selecting the right boot ROM and minor analog audio parameters. SGB/SGB2 require the SGB packet protocol — meaningful additional work; defer.

### Boot ROMs

| Scope choice | Implication |
|---|---|
| **Skip boot ROM** | Initialize CPU and IO registers to documented post-boot state directly. ROMs that rely on boot-ROM-induced state work; ROMs that detect boot-ROM behavior (rare) don't. Simplest to ship. |
| **User-supplied boot ROM** | At runtime, accept a boot ROM file. Require the user to provide one. Full boot-state accuracy when supplied; falls back to skip-boot if absent. |
| **Bundled (open-source) boot ROM** | Include SameBoy's open-source boot ROM (or similar). Boots "for real." No legal concerns since SameBoy's boot ROM is permissively licensed. |
| **Bundled (Nintendo) boot ROM** | Don't. Distributing Nintendo's boot ROM is copyright infringement. |

Recommended: **skip + optional user-supplied**. Bundled SameBoy boot ROM is reasonable if you want a "boots for real" demo out of the box.

### Peripherals / accessories

| Peripheral | Scope choice |
|---|---|
| **Link Cable / Serial** | Stub (return $FF reads, ignore writes) / Full SIO / Multiplayer SIO emulation |
| **Game Boy Camera** | Out of scope / In scope (requires camera-specific MBC and video-input emulation) |
| **Game Boy Printer** | Out of scope / Stub (ack as if printer connected) / Full printer emulation with PNG output |
| **Game Boy Pocket Sonar** | Out of scope (rare) |
| **Mobile Adapter GB** | Out of scope (extremely rare) |
| **Rumble (MBC5+rumble)** | Out of scope / Print debug / Send to host vibration API |
| **MBC7 accelerometer** | Out of scope / Mock (return 0 / center) / Map to host gamepad / Map to host accelerometer |
| **Pocket Camera (Game Boy Camera)** | See above |
| **Transfer Pak (with N64)** | Out of scope |
| **SGB packet protocol** | Out of scope (DMG-only / CGB build) / In scope (SGB/SGB2 build) |
| **SGB border feature** | Out of scope / Render to a wider framebuffer |
| **CGB IR comms (RP)** | Out of scope (very rare game support) |

Recommended default: stub Serial, all others out of scope. Add as games need them.

### Cartridge mappers (MBCs)

Mapper polymorphism (decision #2) is its own ADR. Fidelity scope picks **which mappers are in scope**:

| Scope choice | MBCs supported |
|---|---|
| **Minimum playable catalog** | ROM-only (NROM equivalent), MBC1, MBC3, MBC5 — covers ~80% of catalog |
| **Common catalog** | + MBC2, HuC1, HuC3 — covers ~98% |
| **Full catalog** | + MBC6, MBC7, M161, MMM01, TAMA5, Pocket Camera, Wisdom Tree |

Recommended default: **Common catalog**. Full catalog is mostly long-tail Japanese / educational ROMs; defer until specifically needed.

### APU analog fidelity

| Scope choice | Description |
|---|---|
| **Digital sample-accurate** | Channels output digital values; the mixer sums them. Audio is "correct" in the sense of frequency/duty/envelope. No analog noise modeling. |
| **+ Analog DC bias / charge model** | Model the DC offset of the audio circuit; channel-disable causes a DC step that other emulators handle differently. SameBoy is the gold standard here. |
| **+ Analog filter** | Model the low-pass filter the original hardware applied. Audible difference for some games. |

Recommended default: **digital sample-accurate**. Analog modeling is large additional scope for marginal gain unless audio fidelity is a stated goal.

## Gating mechanism

Once the scope is chosen, decide *how* revision-dependent code paths are gated:

- **Compile-time** (`comptime` parameter; `b.addOptions` flag in `build.zig`): one binary supports one revision. Smallest binary, fastest, simplest code. Cannot switch ROM-by-ROM.
- **Runtime** (field on the console struct): one binary supports all in-scope revisions; the active revision is set when a ROM is loaded based on the cartridge header. Slightly larger binary; no perf penalty if dispatch uses the tagged-union pattern.

Recommended default: **runtime**. Most users prefer one binary that handles any in-scope ROM correctly.

## Save-state schema impact

The active revision must be encoded in the save-state header. Loading a save-state into a console of a different revision is undefined and should fail with a clear error. This is part of decision #4 (save-state schema versioning).

```
header:
  magic: "<EMU>SAVE"
  version: u16
  revision: enum(u8) { dmg, mgb, sgb, sgb2, cgb, agb }
  ... rest of state ...
```

## ADR shape

When grilling with `/grill-with-docs`, the resulting ADR for this repo's fidelity scope should:

1. State the in-scope revisions (e.g., "DMG + CGB").
2. List the *deliberately deferred* items (peripherals, MBCs, SGB protocol).
3. Pick the gating mechanism.
4. Reference this file as the candidate-set documentation.
5. Cite cartridge-header behavior when out-of-scope ROMs are loaded (refuse / fall back to closest in-scope revision / warn).
