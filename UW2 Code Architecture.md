# UW2 Code Architecture

A reference guide to the code structure of Ultima Underworld II: Labyrinth of Worlds, derived from reverse engineering the UW2.EXE disassembly (`uw2_asm.asm`).

## Overview

UW2.EXE is a 16-bit DOS real-mode executable compiled with Borland C. The code is organized into:

- **Fixed code segments** (`segXXX`) — Always resident in memory at predictable addresses. These contain the core engine: rendering, C runtime, input handling, and system-level code.
- **Overlay segments** (`ovrXXX`) — Dynamically loaded/unloaded as needed. These contain game-specific logic: conversations, inventory, spells, menus, save/load, etc. Overlays share memory space and are swapped in on demand.
- **Data segments** (`dseg`, `seg050`–`seg071`) — Static and dynamic game data, string tables, lookup tables, and buffers.

The main data segment is loaded at base address `67d6h` in DOSBox (with the standard debug config).

## Naming Conventions

Functions follow the pattern:

- Code segments: `DescriptiveName_segXXX_XXXX_XXXX` (segment number, segment base, offset)
- Overlays: `DescriptiveName_ovrXXX_XXXX` (overlay number, offset)
- Data references: `DescriptiveName_dseg_67d6_XXXX` (data segment base, offset)

The hex offset suffix is always preserved from the original IDA Pro labels to allow cross-referencing with DOSBox breakpoints.

## Segment Map — Code Segments

| Segment | Lines | Procs | Domain |
|---------|-------|-------|--------|
| seg000 | 45–1318 | 10 | Screen region / dirty rectangle manager |
| seg001_023B | 1322–2117 | — | Memory allocation primitives |
| seg003_0272 | 2360–20349 | 142 | VGA graphics driver, bitmap/sprite drawing |
| seg004_0849 | 20353–35156 | 79 | 3D rendering engine |
| seg005_105F | 35160–46938 | 149 | Borland C runtime library |
| seg006_1413 | 46942–55262 | — | Additional runtime / math support |
| seg007_17A2 | 55266–63402 | — | Additional runtime / math support |
| seg008_1B09 | 63406–66870 | — | Additional runtime / math support |
| seg009 | 66874–68111 | 11 | Cursor/sprite rendering, texture page access |
| seg010_1CA1 | 68115–69223 | — | Click area / event registration |
| seg011 | 69227–70403 | 4 | Game loop, joystick input |
| seg012_1D31 | 70407–70642 | — | Small utility segment |
| seg013_1D3C | 70646–71281 | — | Small utility segment |
| seg014 | 71285–71981 | 6 | Options UI grid, click areas |
| seg015_1D7C | 71985–76161 | — | Display locking, interrupt management |
| seg016_1E73 | 76165–85706 | — | Drawing primitives, filled shapes |
| seg017_2179 | 85710–86978 | — | String/text rendering |
| seg018_21b4 | 86982–87109 | — | Small utility segment |
| seg019_21BA | 87113–91017 | — | Extended runtime support |
| seg020 | 91021–91188 | 1 | EGA planar tile blit |
| seg021_22FD | 91192–95639 | — | Decompression, stack switching |
| seg022_2405 | 95643–97997 | — | Game object processing |
| seg023 | 98001–98415 | 2 | Palette cycling / lighting |
| seg024_24E9 | 98419–104557 | — | Game world processing |
| seg025_26A1 | 104561–106206 | — | Object type handling |
| seg026_2716 | 106210–110430 | — | Movement, collision |
| seg027_2856 | 110434–113213 | — | NPC/mobile processing |
| seg028_2941 | 113217–117616 | — | AI, pathfinding |
| seg029_2A8E | 117620–121785 | — | Object lookup, reverse mapping |
| seg030_2BB7 | 121789–125607 | — | World interaction |
| seg031_2CFA | 125611–130587 | — | Combat, damage processing |
| seg032_2E9B | 130591–134302 | — | Timer, render timestamp sync |
| seg033_2FBE | 134306–137772 | — | Rendering support |
| seg034_310D | 137776–139948 | — | Map/tile rendering |
| seg035_31AB | 139952–143062 | — | Object rendering |
| seg036_32A9 | 143066–143456 | — | Small utility segment |
| seg037_32C0 | 143460–148268 | — | Player update tick |
| seg038_342C | 148272–148988 | — | Small utility segment |
| seg039_3452 | 148992–151309 | — | Input processing |
| seg040_34E7 | 151313–154710 | — | Sound/music management |
| seg041_35D7 | 154714–155221 | — | Small utility segment |
| seg042_35ED | 155225–155996 | — | Small utility segment |
| seg043_3619 | 156000–157951 | — | Text display region management |
| seg044_368F | 157955–162195 | — | UI panel drawing |
| seg045_379C | 162199–163049 | — | Small utility segment |
| seg046_37CD | 163053–167372 | — | Extended UI support |

## Segment Map — Overlay Segments

| Overlay | Lines | Procs | Domain |
|---------|-------|-------|--------|
| ovr092 | 387626–387762 | 7 | Conversation/trade stubs |
| ovr093 | 387766–391243 | 17 | ARK file I/O, block headers |
| ovr094 | 391247–397345 | 34 | Automap system |
| ovr095 | 397349–404288 | 62 | NPC conversation VM |
| ovr097 | 409184–417509 | 46 | Trading / bartering |
| ovr099 | 417519–417644 | 7 | Stub placeholders |
| ovr101 | 417654–423146 | 16 | Character generation |
| ovr103 | 423629–429228 | 38 | Conversation UI, portrait data |
| ovr104 | 429232–429816 | 7 | Cutscene variants, critter data loading |
| ovr108 | 436475–446984 | 79 | Cutscene engine |
| ovr109 | 446988–447067 | 2 | Input event handler registration |
| ovr110 | 447071–460909 | 96 | World events, tile changes, traps, pit fighting |
| ovr112 | 460919–462867 | 23 | Game initialization, main loop, shutdown |
| ovr114 | 466052–466561 | 4 | Error handling, fatal messages |
| ovr116 | 466571–467809 | 6 | LZW compression, screenshot capture |
| ovr117 | 467813–468845 | 5 | Critter animation cache (LRU) |
| ovr118 | 468849–470633 | 20 | Graphics file I/O, palette fade/transitions |
| ovr119 | 470637–473980 | 26 | Art/texture asset loading |
| ovr121 | 474189–478145 | 15 | Inventory, container stack |
| ovr122 | 478149–480003 | 14 | Player data save/load, object copy buffer |
| ovr123 | 480007–481703 | 13 | Spell casting, rune bag UI |
| ovr124 | 481707–485409 | 23 | Inventory slot management |
| ovr125 | 485413–491206 | 22 | Item details, pick up/place, hit testing |
| ovr128 | 497071–498014 | 9 | Level data (lev.ark), tile map |
| ovr134 | 498311–498723 | 4 | Object data loading, credits |
| ovr136 | 500811–503666 | 34 | Options/settings menu UI |
| ovr137 | 503670–504861 | 10 | Stats panel, mode switching |
| ovr139 | 512009–513903 | 9 | Message scroll, [MORE] prompt, text output |
| ovr140 | 513907–514662 | 8 | Texture/terrain loading |
| ovr142 | 514672–517514 | 16 | Player attributes, armour |
| ovr143 | 517518–521048 | 20 | Spells, camera control, compass, movement |
| ovr146 | 521064–521084 | 1 | Stub placeholder |
| ovr147 | 521088–523945 | 11 | Main menu, menu item rendering |
| ovr149 | 523976–527168 | 18 | Save/load game |
| ovr151 | 527539–529530 | 18 | SCD scripting engine |
| ovr154 | 530489–537473 | 35 | Spell effects (healing, detection, etc.) |
| ovr162 | 551586–551627 | 1 | Trigger object data loading |
| ovr165 | 553382–553433 | 2 | Full-screen UI dispatch stubs |
| ovr167 | 564404–566926 | 27 | File I/O, save encryption, utilities |

## Major Systems

### 3D Rendering Engine (seg004_0849)

79 functions implementing a complete software 3D renderer:

- **Render VM**: A bytecode interpreter that processes 3D model description scripts. Opcodes handle vertex projection, conditional branching (dot product sign tests), subroutine calls, and viewer-relative computations.
- **3x3 Matrix Math**: Euler angle rotation matrices built from yaw, pitch, and roll, applied to vertices and view matrices.
- **Polygon Clipping**: Two complete Sutherland-Hodgman clipping pipelines — one in model space (4 frustum planes) and one in screen space (4 viewport edges), plus a textured polygon clipping pass with UV interpolation.
- **Perspective Projection**: Vertices transformed from 3D to screen space with self-modifying code for performance.
- **Vertex Lighting**: Per-vertex lighting with distance attenuation using a Newton-Raphson integer square root.
- **Texture-Mapped Rasterization**: Dual-axis scanline rasterizers — Y-axis for floors/ceilings and X-axis for walls — with edge gradient calculations for affine texture mapping.
- **Texture Decompression**: RLE decode with support for 4bpp, 5bpp, and 8bpp formats, plus palette remap tables.
- **EMS Memory Management**: Expanded memory page mapping with an LRU cache for creature graphics.

### VGA Graphics Driver (seg003_0272)

142 functions for low-level VGA graphics:

- Direct VGA register manipulation (ports 3C4h/3CEh/3D4h)
- Bitmap and sprite drawing with clipping
- Palette management and color operations
- Screen buffer management and blitting

### C Runtime Library (seg005_105F)

149 functions — the Borland C RTL compiled into the executable:

- **Heap management**: Separate near heap (stack-relative, sbrk-based) and far heap (DOS memory blocks) with malloc/free/realloc, block coalescing, and free list management.
- **Buffered file I/O**: Full stdio implementation — fopen/fclose/fread/fwrite/fgets/fputc/fflush with 20-entry file table, dirty buffer tracking, and text/binary mode support.
- **Formatted I/O**: Complete sprintf engine (with %d, %s, %x, %p, %f format specifiers) and sscanf parser (22-case format dispatch).
- **String operations**: strcmp, strrchr, strnicmp, strlen, memcpy, toupper/tolower.
- **Date/time**: Unix epoch conversion, localtime equivalent, DST calculation, DOS date/time packing.
- **Math**: Far pointer arithmetic (add/subtract/compare with segment normalization), 32-bit multiply/divide.
- **Program lifecycle**: atexit registration, shutdown dispatch, abnormal termination handler.

### Screen Region Manager (seg000)

10 functions implementing a dirty rectangle system:

- Up to 64 screen regions tracked in 16-byte slot structures
- Each region has position, size, draw mode, callback, and group assignment
- Dirty flag propagation with rectangle intersection testing across groups
- Interrupt-safe redraw dispatch (CLI/STI around drawing operations)

### Cutscene Engine (ovr108)

79 functions for in-game cinematics:

- Frame sequencing and animation playback
- Viewport scrolling and scene transitions
- Cutscene-specific rendering and palette control

### Conversation VM (ovr095)

62 functions implementing a bytecode virtual machine for NPC dialogue:

- Stack-based VM with arithmetic ops (ADD, SUB, MUL, DIV, MOD, AND, OR)
- Comparison operators (GT, GE, LT, LE, EQ, NE) and conditional branching
- String operations: Random, Compare, Plural, Contains, Append, Length, Val
- SAY and RESPOND opcodes for dialogue display
- Memory management with custom allocator (init/alloc/free/realloc) and boundary tag coalescing
- Imported function calls and variable binding

### Automap System (ovr094)

34 functions for the dungeon automap display:

- Tile-based map rendering with wall adjacency detection in 4 cardinal directions
- Dithered wall lines with RNG-based color variation
- Door indicators and diagonal wall segments
- User-placed map notes with click-to-select (nearest note calculation)
- World portal navigation — 8-world gem with visited/current status icons
- Level switching with map note persistence to lev.ark
- Map piece octant calculation for the map piece item

### Inventory System (ovr121, ovr124, ovr125)

60 functions across three overlays:

- **Container stack**: Open/close containers with a linked list stack, back-navigation, and scrolling
- **Slot management**: 19+ inventory/paperdoll slots with hit testing from screen coordinates
- **Object interaction**: Pick up, place, swap, combine objects in slots with weight tracking through container chains
- **Inventory redraw**: Selective slot redrawing, panel mode switching

### Trading / Bartering (ovr097)

46 functions for NPC trading:

- 2x3 grid layout for player and NPC trade slots with mouse hit testing
- Left-click (place item) and right-click (examine) handlers for both sides
- Item valuation: `GetTrueValueOfItemToNPC` with like/dislike modifiers
- Offer/demand/decline/judgement dialogue flow
- Object transfer between player and NPC inventories

### Options Menu UI (ovr136)

34 functions implementing the in-game options panel:

- Menu panel data structure with function pointers at `[si*4+6]`
- Secondary callbacks at `[si*2+30h]`, callback arguments at `[si*2+22h]`
- Save/restore game prompts and execution
- Sound and music toggle menus
- Detail level settings
- Mouse-to-item hit testing and selection highlighting

### SCD Scripting Engine (ovr151)

18 functions for Scheduled Command Data:

- SCD rows sorted by timestamp, executed in order
- Level-based block indexing (level number << 2 into offset table)
- Row insertion with sorted merge and optional immediate execution
- Dirty flag tracking for reload detection
- Compression support for SCD data

### World Events System (ovr110)

96 functions — the largest overlay — handling dynamic world state:

- Tile height changes with object z-position adjustment
- Trap execution and trigger processing
- Switch/lever activation via `FindAndUseSwitch`
- Pit fighting arena bounds checking
- Terrain modification from scripts

### Art / Texture Asset Loading (ovr119)

26 functions for game art resources:

- Art file header parsing with offset tables (count * 32 bytes)
- Paged texture memory allocation (0x4000 byte pages)
- Texture entry storage with combined page:offset encoding
- Object icon loading from .gr files with optional base offset
- RLE/raw art unpacking into allocated memory
- Individual art image loading to specific icon slots

### Graphics File I/O & Palette (ovr118)

20 functions for graphics resource management and visual effects:

- Bitmap and font loading from data files
- Palette loading from pals.dat
- Timed palette fade-in and fade-out (iterative RGB interpolation)
- Screen transition effects: color map and shade table variants
- Forward and reversible (save/restore framebuffer) transition modes

### Player Data Save/Load (ovr122)

14 functions for player state persistence:

- Object copy buffer with 8-byte slot records
- Inventory slot remapping during copy and restore operations
- Player object serialization (0x1B bytes) with linked list traversal
- Paperdoll and object-in-hand state preservation

### Save File Encryption (ovr167)

27 functions including save file security:

- XOR-based encryption/decryption for player.dat
- Key schedule generation from a seed byte across 5 loops with different stride multipliers (191 key bytes total)
- Direction/heading calculations (vectors to compass directions)
- Directory verification for required game data folders
- General-purpose save-data-to-file wrapper

### Game Initialization & Main Loop (ovr112)

23 functions managing the game lifecycle:

- Command-line parsing and DATA folder setup
- Splash screen sequence (part 1 and part 2)
- Game loop bitfield state management (`GameLoopBitField` at `dseg_67d6_5DAC`)
- Screen mode transitions (gameplay, conversation, fullscreen UI)
- Action timer reset, render timestamp synchronization
- Shutdown and cleanup sequence

### LZW Screenshot Capture (ovr116)

6 functions implementing GIF-style LZW compression:

- Hash-table dictionary (0x138B entries) with clear/EOI codes
- Variable bit-width code output with byte boundary handling
- Raster-order screen pixel reading (320x200)
- Complete compress-to-file pipeline

### Character Generation (ovr101)

16 functions for creating a new player character:

- Mouse-driven option selection on chargen screens with grid-based hit testing
- Skill rolling and stats display
- Player initialization and new game startup wrapper
- Equipment setup with null-guarded slot assignment

### Critter Animation Cache (ovr117)

5 functions managing creature sprite memory:

- LRU eviction policy for animation cache slots (256 entries)
- Stale slot release with status flag tracking
- Batch allocation of N slots for creature loading
- Two-pass allocation with duplicate detection

## Key Data Segment Variables

| Offset | Name | Description |
|--------|------|-------------|
| `dseg_67d6_2256` | `CurrObj` | Currently processed game object |
| `dseg_67d6_828E` | `PlayerObject` | The player object |
| `dseg_67d6_8292` | `DungeonLevel` | Current dungeon level number |
| `dseg_67d6_6B0C` | `ObjectInHand` | Object currently held by cursor |
| `dseg_67d6_5D68` | `MaybeQuitGameLoop` | Quit game loop flag |
| `dseg_67d6_5DAC` | `GameLoopBitField` | Game loop state/mode flags |
| `dseg_67d6_1622` | `PaperDollArray` | Open container linked list |
| `dseg_67d6_2194` | Texture lookup table | Page:offset entries for textures |
| `dseg_67d6_36F9` | Map notes dirty flag | Automap notes need saving |
| `dseg_67d6_8634` | SCD data pointer | Scheduled Command Data |
| `dseg_67d6_1BF4` | atexit callback count | C runtime atexit table |
| `dseg_67d6_1D06` | File table (20 entries) | C runtime stdio file table |
| `dseg_67d6_34B0` | `WriteTextRelated` | Message scroll state |
| `dseg_67d6_2506` | `IsPlayerDoingSomething` | Player action state flag |
| `dseg_67d6_82A0` | Compass click area handles | Viewport directional button handles |
| `dseg_67d6_33C8` | Player distance | Distance from camera to player |
| `dseg_67d6_33CC` | Player heading | Heading angle from camera to player |
| `dseg_67d6_773` | `MotionInput` | Current movement direction codes |
| `dseg_67d6_77B` | Control mode flag | Joystick/mouse control mode selector |
