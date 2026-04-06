# UW2 Cutscene Engine Reverse Engineering Notes

## Overview

The UW2 cutscene engine (ovr108, 79 functions) plays scripted animations using a bytecode command system. Cutscene data is stored in the `CUTS/` directory using the Deluxe Paint LPF (Large Page File) animation format.

## File Structure

Each cutscene has:
- **CSxxx.N00** — Bytecode control file (command stream)
- **CSxxx.N01, N02, ...** — LPF animation files (DPaint format)
- **LBACK*.BYT** — Raw 320x200 panorama background bitmaps (UW2 only)

### LPF Format (Deluxe Paint Animation)

Standard EA Deluxe Paint animation format:
- **Offset 0x00-0x7F**: 128-byte header (magic "LPF ", dimensions, frame count, fps, hasLastDelta flag)
- **Offset 0x80-0xFF**: 128-byte color cycling block (16 x 8-byte CRNG entries)
- **Offset 0x100-0x4FF**: 1024-byte palette (256 entries, BGR + padding)
- **Offset 0x500-0xAFF**: Large page descriptor table (256 x 6 bytes)
- **Offset 0xB00+**: Large page data (frame data)

Key header fields:
- **0x1A** `hasLastDelta` (byte): If non-zero, the last frame is a loop-back delta (resets buffer to frame 0). This frame should NOT be displayed. `FinalPixelBuffer` for delta chaining must be captured BEFORE this frame is decoded.
- **0x40** `nFrames` (dword): Total frame count including the loop delta if present.
- **0x44** `framesPerSecond` (word): Authored playback rate from DPaint.

### Color Cycling Block (offset 0x80)

16 entries of 8 bytes each (IFF CRNG format):
```
Offset 0-1: count (reserved)
Offset 2-3: rate (16384 = 60 steps/sec)
Offset 4-5: flags (bit 0: active, bit 1: reverse)
Offset 6:   low palette index
Offset 7:   high palette index
```

### Delta Encoding

Frames use Run/Skip/Dump RLE compression. Each frame is delta-encoded against the previous frame's pixel buffer. The buffer persists across file switches (N01->N02->N03 etc).

## Cutscene Command System (28 commands)

| Cmd | Name | Params | Description |
|-----|------|--------|-------------|
| 0 | show-text | color, string | Display subtitle using palette[color] |
| 1 | set-flag | - | Clears animation flag at +39h |
| 2 | no-op | 2 | No operation |
| 3 | pause | duration | Pause for arg/2 seconds |
| 4 | to-frame | count, ? | Play N frames inline |
| 5 | frame-set | - | Segment boundary; frame field = frame count |
| 6 | end-cutsc | - | End cutscene |
| 7 | rep-seg | count | Repeat segment N times |
| 8 | open-file | csNo, extNo | Load new LPF file |
| 9 | fade-out | rate | Fade to black (higher=faster, 0=instant) |
| 10 | fade-in | rate | Fade from black |
| 11 | frame-trigger | target | Conditional frame advance |
| 12 | bit-control | value | Sets flag bit 4 to (arg & 1) |
| 13 | text-play | color, string, audio | Show text + play VOC (999=no audio) |
| 14 | wait-secs | secs, ? | Wait N seconds |
| 15 | klang | - | Play klang sound (UW1 only) |
| 16 | pal-copy | src, dst, count | Copy palette data |
| 17 | timer-cb | func, delay, param | Register timer callback |
| 18 | no-op | 4 | No operation |
| 19 | pal-interp | target, speed, frames | Interpolate palette toward PALS.DAT[target] |
| 20 | viewport-setup | w, h, offset | Set panorama canvas dimensions |
| 21 | set-start | x, y | Set scroll start position |
| 22 | map-file | pos, extent, fileIdx | Load LBACK[fileIdx].BYT for panorama |
| 23 | start-scroll | idx, delta, ? | Begin scrolling with direction table lookup |
| 24 | audio-setup | fileNo | Set audio file (999=none, else decrement) |
| 25 | music | theme | Play XMI music theme |
| 26 | no-op | 1 | No operation |
| 27 | audio-wait | timeout | Wait for audio completion |

## Func 19: Palette Interpolation (NOT rotation)

Confirmed via pixel-perfect DOSBox capture comparison: func 19 interpolates the LPF's embedded palette toward a target palette from PALS.DAT. Each step moves each color channel by 4 (VGA DAC granularity: 6-bit values mapped to 8-bit).

- `param[0]` = target palette index in PALS.DAT
- `param[1]` = speed (interpolate every N frames)
- `param[2]` = total frames

Only palette entries within the LPF's color cycling ranges are interpolated. The cycling ranges come from the LPF header's 128-byte block at offset 0x80.

## Scroll Direction Table (dseg_67d6+0x1068/0x1070)

The `start-scroll` command's first parameter is a table INDEX, not a raw pixel speed:

| Index | X | Y | Direction |
|-------|---|---|-----------|
| 0 | 0 | +1 | Down |
| 1 | +1 | 0 | Right |
| 2 | 0 | -1 | Up |
| 3 | -1 | 0 | Left |

Actual speed = delta (param[1]) * table[index]. The scroll position formula:
```
X = start_X + frame * delta * X_table[index]
Y = start_Y + frame * delta * Y_table[index]
```

## Panorama Scrolling Architecture

The engine uses a 640-pixel-wide VGA offscreen buffer for hardware scrolling:

1. `map-file` loads LBACK*.BYT bitmaps into the 640-wide buffer (e.g. LBACK000 at x=0, LBACK001 at x=320)
2. `SetViewportFar` changes VGA CRT start address registers (0x0C/0x0D) to pan across the buffer
3. `DrawArtToScreen` renders the current LPF animation frame into the buffer
4. Fine pixel panning via VGA register 0x3C0 index 0x33

### File Roles (consistent pattern for all scroll scenes)

| Role | Horizontal | Vertical |
|------|-----------|----------|
| Pre-scroll animation (has animated element) | N02 (cart) | N05 (flags) |
| Sprite overlay during scroll | N03 (cart sprite) | N06 (flag sprites) |
| Post-scroll clean scene (no animated element) | N04 | N07 |
| Panorama background bitmap | LBACK000/001 | LBACK002/003 |

## BYT.ARK Splash Screens

The `DATA/BYT.ARK` archive contains compressed 320x200 screens. From disassembly (`SplashPart1_ovr112_36`):

| Entry | Content | Palette (PALS.DAT) |
|-------|---------|-------------------|
| 6 | Origin "Presents" logo | 5 |
| 7 | Looking Glass Technologies logo | 6 |

The Godot project's `bytloader.cs` had incorrect palette indices (15) for these entries.

## Flag Byte (+5Bh)

| Bit | Purpose |
|-----|---------|
| 0 | Pause flag |
| 1 | Animation trigger (set by cmd 11) |
| 2 | Animation active (cleared by cmd 5/6) |
| 3 | End marker (cleared by cmd 6 only) |
| 4 | State flag (set by cmd 12) |
| 5 | Audio/palette skip flag |
| 7 | Fade flag |

## Intro Sequence

The UW2 intro plays three cutscenes in sequence:
- **CS000**: Lord British's letter, panorama scrolls, feast, fireworks, "But the next morning..."
- **CS001**: Dawn castle scene, blackrock dome descending
- **CS002**: Guardian's appearance (triggered after meeting Lord British in-game, NOT part of intro)

CS011 = title screen animation ("Ultima Underworld II / Labyrinth of Worlds")
CS012 = credits/acknowledgements

Note: CS000 references N11 in its bytecode, but N11 contains unfinished art that was cut from the final game.
