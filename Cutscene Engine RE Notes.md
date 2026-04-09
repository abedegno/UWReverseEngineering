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

16 entries of 8 bytes each (IFF CRNG format). **Actively used by UW2** — parsed by
`ReadAnimationHeader_ovr108_751` (line 438205) and applied per-frame by
`UpdatePaletteFadeTimers_ovr108_934` (line 438583) which calls
`RotatePaletteEntry_seg023_9` (line 98009) for each range with rate > 0.

```
Offset 0-1: counter/accumulator (big-endian)
Offset 2-3: rate — added to counter each tick; rotation when counter >= 65 (big-endian)
Offset 4-5: flags (big-endian; UW2 appears to ignore these — active if rate > 0)
Offset 6:   low palette index
Offset 7:   high palette index
```

Rotation algorithm (`RotatePaletteEntry_seg023_9`, line 98009): save first RGB entry,
shift all entries backward by one position, place saved entry at end (forward rotation).

Only CS011 (title screen) has active CRNG ranges:

| Range | Indices | Count | Rate | Effect |
|-------|---------|-------|------|--------|
| CRNG 0 | 57-65 | 9 | 18 | ~5.0 rotations/sec — main flame |
| CRNG 1 | 54-56 | 3 | 16 | ~4.5 rotations/sec — shimmer |
| CRNG 2 | 43-48 | 6 | 24 | ~6.7 rotations/sec — fast accent |
| CRNG 3 | 49-51 | 3 | 14 | ~3.9 rotations/sec — slow pulse |
| CRNG 4 | 52-53 | 2 | 24 | ~6.7 rotations/sec — fast flicker |

Rotation rates from DOSBox frame capture analysis (70fps capture, frames 425-750):
CRNG 0 (rate 18) rotates 1 step every 14 capture frames = 5.0 steps/sec.
CRNG 3 (rate 14) rotates 1 step every 17 capture frames = 4.1 steps/sec.
At PIT timer rate 18.2 Hz with threshold 65: `rate/65 * 18.2` matches observed rates.

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

## Func 19: Palette Interpolation

Note: UW2 uses BOTH palette interpolation (func 19, for smooth colour transitions)
AND CRNG palette rotation (from LPF header, for animated flame effects). They are
independent systems — interpolation is command-driven, cycling is data-driven from
the LPF file's CRNG block.

Func 19 (`Cutscene_19_Unk_ovr108_1229`, line 440603) interpolates the LPF's embedded
palette toward a target palette from PALS.DAT using **linear interpolation**.

- `param[0]` = target palette index in PALS.DAT (loaded via `OpenPalsData`, line 440668)
- `param[1]` = speed (timer delay — interpolate every N frames)
- `param[2]` = total frames (stored in `[bx+5934h]`, line 440656)

### Interpolation Formula

From `InterpolatePaletteRange_ovr108_32CC` (line 446461):
```
For each palette byte (R, G, B across all entries):
    result = source_byte + (current_step * (target_byte - source_byte)) / total_steps
```

This is standard linear interpolation, NOT step-by-4 as previously documented.

- Source palette: snapshot of LPF embedded palette at time func 19 fires
  (copied to `[si+55CAh]`, lines 440689-440699, 768 bytes = 256 * 3)
- Target palette: loaded from PALS.DAT to `dseg_67d6_59F4` (line 440664)
- Range: ALL 256 entries (`[bx+5974h]` = 0x100, line 440656), not just cycling ranges
- Timer-driven: `RegisterTimerCallback` (line 440629) with `stub108_133` →
  `ApplyTimerPaletteTransition_ovr108_3241` (line 446364)

### Usage

- CS000 N13: `pal-interp [8, 4, 80]` — sunset-to-night over 80 frames, every 4th frame
- CS001 N04: `pal-interp [9, 3, 9]` — dawn colour shift over 9 frames, every 3rd frame

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

### Panorama Rendering Pipeline (from disassembly)

When panorama mode is active (`[si+4Fh] != 0`, set by func 20 when canvas != 320x200),
the normal `DrawBitMap` call is **skipped** (ovr108_24AB, line 443935). Instead:

1. `AnimateViewportScroll` (ovr108_B8E, line 439098) shifts the VGA CRT start
   address via `SetViewportFar`, panning the LBACK panorama.
2. The LPF frame is decoded into the persistent pixel buffer (standard delta chaining).
3. `DrawArtToScreen` (line 444058) draws the decoded frame at a **fixed VGA screen
   position** `([si+13h], [si+15h])` — both appear to be 0 (never explicitly written).
4. The draw height is `200 - (vpOffsetY + 1)` = 159 pixels (scene area only).

**Critical insight:** The sprite is drawn at a fixed screen position, NOT into the
scrolling panorama buffer. The LPF animation's pixel positions already compensate
for the scroll progression — e.g., flag writes move upward in the sprite's
coordinate space to maintain a fixed screen position as the viewport pans.

### Sprite Overlay Approach

The LBACK background already contains the animated elements at rest (flags on poles,
cart in street). The LPF sprite files (N03, N06) contain only the **animated deltas**
— small patches (~300-400 pixels per keyframe) that update one animated element at
a time (flags take turns waving, cart moves behind bridge).

To simulate the original rendering:
1. Decode the sprite LPF with LBACK raw pixels as the base buffer, resetting to
   LBACK before each keyframe (`is_sprite` mode). This ensures RLE skip areas
   contain LBACK pixels and prevents accumulation from prior keyframes.
2. Crop the viewport from the LBACK composite (panorama scroll).
3. Overlay **only the RLE-written pixels** (via write mask) onto the cropped viewport
   at fixed screen positions. This preserves the scrolling background while drawing
   animated elements at their intended screen locations.

### Subtitle Bar (vpOffsetY)

The viewport-setup command (func 20) third parameter `vpOffsetY` reserves space for
subtitles when a panorama is active (canvas != 320x200):
- Scene height = `200 - vpOffsetY` pixels (e.g., 160px with vpOffsetY=40)
- Subtitle bar = bottom `vpOffsetY` pixels (black background)
- Text positioning from `RenderCutsceneText_ovr108_157B` (line 441321):
  - Line spacing = font height (TextPos[bx+6], line 441390)
  - Bottom margin = 2px (from `inc dx; inc dx` at ovr108_15CE, lines 441401-441403)
  - Text rendered bottom-to-top, bottom-aligned in the subtitle area

### Font

Cutscene subtitles use **FONTBIG.SYS** (font index 3), hardcoded in the engine
via `OpenFont(3)` at ovr108_2E1A (line 445603). Not bytecode-driven.
- Height: 15px (header offset 6)
- No inter-character spacing — glyph width used directly (no +1 pixel gap)
- Word wrapping at 320px (full display width)

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

## Segment Boundaries and Frame-Set

The `frame-set` command (func 5) terminates a segment. Its frame field is the segment
length — the frame loop runs `range(seg_frames)`, so frames 0 to seg_frames-1 are
displayed. Commands at the frame-set frame number (e.g. fade-out at frame 24 in a
24-frame segment) fire **after** the last displayed frame, without displaying an
additional frame. This matches the DPaint LPF format where the frame count does not
include the boundary frame.

These boundary commands typically set up state (e.g. fade-out) that carries over into
the next segment. If a fade-out is still active when a new segment starts, it should
complete before the new content is rendered.

## Open-File and Auto-Advance

The cutscene VM auto-advances to the next LPF file at each segment boundary
(ext increments by 1 in decimal, which maps correctly through the octal filename
encoding). The `open-file` command (func 8) overrides this by loading a specific file
mid-segment and setting the current extension.

When `open-file` appears at the frame-set boundary frame, it fires after the last
displayed frame and prevents auto-advance for the next segment.

### CS000 N11 Skip Pattern

CS000's bytecode uses `open-file` commands to skip N11, which contains unfinished art
from an earlier build. The sequence:
- Seg 8 plays N10, then `open-file [0, 10]` jumps to N12 (skipping N11)
- Seg 9 plays N12, then `open-file [0, 8]` jumps back to N10
- Seg 10 plays from N10, then `open-file [0, 11]` jumps to N13
- N11 is never reached through normal playback

## Scroll State Across Segments

The panorama scroll (`start-scroll`, func 23) is segment-scoped: it only applies to
segments that contain a `start-scroll` command. Segments without `start-scroll` render
LPF frames directly, even if the panorama composite still exists.

The panorama composite itself persists across segments until `viewport-setup` (func 20)
fires, which clears all panorama state. This means:
- Pre-scroll segments: show LPF frames directly (composite exists but isn't used for display)
- Scroll segments: crop viewport from composite, overlay sprites via write masks
- Post-scroll segments (no new start-scroll): show LPF frames directly

## Timer-Driven Scroll

The actual scroll position is driven by `UpdateViewportFromTimer_ovr108_3333`
(line 446546), a timer callback registered by func 23 via `RegisterTimerCallback`
(line 440629). `AnimateViewportScroll_ovr108_B8E` (line 439098) also modifies the
viewport position each frame, but `seg049_3EE2_18` (the frame counter it uses) is
never incremented — it stays at 0. The timer callback uses separate state variables
at `[bx+5914h]`, `[bx+5934h]`, `[bx+5954h]`, `[bx+5974h]`.

## Cutscene Inventory

| File | Trigger | Content |
|------|---------|---------|
| CS000 | Intro sequence | Lord British's letter, panorama scrolls, feast, fireworks |
| CS001 | Intro sequence | Dawn/dusk transitions |
| CS002 | In-game | Stained glass window, panorama scroll, victory text |
| CS004-007 | Unknown | Stub files (N00 only, no LPF files) |
| CS011 | Title screen | "Ultima Underworld II / Labyrinth of Worlds" with CRNG flame effect |
| CS012 | End game | Acknowledgements |
| CS030-036 | Sleep (sleep.cs:336) | Dream/vision sequences |
| CS040 | Unknown | Short sequence |
| CS403 | Death | Death animation (alpha channel sprites) |

## Intro Sequence

The UW2 intro plays CS000 then CS001 in sequence.
CS002 is triggered in-game (NOT part of the intro).
