# 20 — Porting Decompiled Source

The highest-fidelity route: you compile the game's own code for ARM and replace the
console underneath it.

Reference: `Other Ports/melee-Vita-vita-port` — Super Smash Bros. Melee (GameCube)
built from [doldecomp/melee](https://github.com/doldecomp/melee) for Vita and PS5,
release 0.8.15.

---

## 1. Architecture rule

```
 ┌──────────────────────────────────────────────────┐
 │ src/melee/**  src/sysdolphin/**   UNCHANGED      │  game logic
 └─────────────────────┬────────────────────────────┘
                       │ Dolphin SDK calls
 ┌─────────────────────▼────────────────────────────┐
 │ platforms/vita/game/  — NEW native layer         │
 │  gx.c gx_render.c   GX  -> GXM                   │
 │  dvd.c              disc -> ISO streaming        │
 │  audio.c            AX/ARAM -> SceAudioOut       │
 │  card.c             memory card -> file          │
 │  pad.c vi.c os.c thp.c                           │
 └─────────────────────┬────────────────────────────┘
                       ▼  SceGxm / vita2d / vitaShaRK
```

**Do not edit game code.** Everything platform-shaped is swapped underneath, so the
decompilation stays diffable against upstream. The moment you start patching game
source, you lose the ability to merge upstream fixes.

## 2. ABI flags (the part that silently breaks everything)

| Flag | Reason |
|------|--------|
| `-fno-short-enums` | ARM EABI defaults to 1-byte enums; the original console ABI used 4 bytes. On-disc structs and vtable layouts break otherwise. |
| `-fgnu89-inline` | matches the original compiler's inline linkage |
| `-fno-strict-aliasing` | game code type-puns constantly |
| `-fwrapv` | signed overflow semantics of the original toolchain |
| `-ffp-contract=off` | stop FMA contraction changing float results |
| `-Wno-scalar-storage-order` | silences warnings when using the big-endian attribute |

`-fno-short-enums` alone rescues entire categories of "the game state is garbage"
bugs.

## 3. Big-endian data on a little-endian CPU

Two strategies:

**A. Compiler route** (Melee): GCC `scalar_storage_order("big-endian")` on structs
mirroring on-disc layouts. Zero runtime cost, needs GCC ≥ 15, and every struct must
be annotated correctly — one missed field is a silent corruption.

**B. Byte-swap route**: swap on load into native structs. One pass over the data,
portable, and much easier to verify. Preferred unless you need zero-copy access to
on-disc structures.

Decide per subsystem, not per project. Melee streams from the user's ISO and needs
zero-copy, so A. A port that converts assets offline should use B everywhere.

## 4. Subsystem replacement checklist

| Console system | Replacement | Difficulty |
|----------------|-------------|-----------|
| GPU (fixed-function or GX) | GXM renderer + shader cache | hardest |
| disc/optical | ISO/archive reader with async thread | medium |
| audio DSP | software mixer into `SceAudioOut` | medium |
| memory card | file-backed container emulating the FS | easy |
| controller | `SceCtrl` + remap table | easy |
| video playback | software decode or transcode to H.264 | medium |
| OS threads/alarms | pthread + a per-frame poll | easy but easy to get wrong |

## 5. Order of work that actually succeeds

1. get the game to **link** against stubs (a build with no platform layer)
2. make `main` run and return
3. bring up VI/display with a clear colour
4. bring up the renderer on one known scene
5. audio
6. input
7. saves
8. performance

Do not start at 8. Every port that tried burned out.

## 6. Renderer specifics (GX → GXM in one paragraph)

Hash the render state that affects compiled code (textures, combiners, lighting,
fog, indirection), look it up in a RAM table, then a writable cache, then a packaged
cache, then generate Cg and compile via `vitaShaRK`. Cache the resulting program.
Store `{magic, size, hash, kind, program}` records in a versioned file
(`warm5.bin`). See docs 04/05 and `platforms/vita/game/gx_render.c`.

## 7. Build & release discipline

- one build script with `-Configuration Release|Debug`
- Release = `-O3 -DNDEBUG`, no logging, no debugger, no symbols
- diagnostics behind explicit switches (render trace, cache sealing, resolution
  scale)
- version in a single `version.json`, consumed by the build, the VPK name, and the
  update feed
- trained shader cache packaged into the VPK
- signed update manifest if you ship updates

## 8. When this route is not available

If there is no decompilation and no source, this route is closed — and so is the
`.so` route, unless the binary happens to be ARM. That leaves emulation (only viable
for far weaker hardware) or a framework reimplementation. Be honest with yourself
early: see doc 26.

## 9. Sources

- `Other Ports/melee-Vita-vita-port/README.md`, `platforms/vita/README.md`
- `platforms/vita/CMakeLists.txt` (the flag block, `-fno-short-enums` rationale)
- `platforms/vita/game/*.c` (every subsystem replacement)
