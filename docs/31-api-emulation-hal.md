# 31 — Cross-Platform API Emulation (HAL Layer)

Instead of porting a game, port the *platform it was written for*. Reimplement its
API on the Vita and compile the game unchanged.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/api-emulation.md`, MIT). See [CREDITS](../CREDITS.md).

---

## 1. When this beats a rewrite

| Condition | Why it works |
|-----------|--------------|
| game targets another handheld/console with a documented API | the call surface is finite |
| game code is nearly unchanged | only the platform layer differs |
| the target GPU API can express the source's fixed-function pipeline | shaders are generatable |
| a reimplementation would be more work than the emulation layer | usually true for big games |

Reference: mkwiita — a 1:1 Mario Kart Wii recreation built for 3DS (libctru/citro3d),
ported to Vita with ~2,200 lines of platform layer across 7 files and the game code
untouched.

## 2. Architecture

```
Game (unmodified)
   ├── <platform> headers → platform/vita/include/<platform>.h    reimplemented
   └── game source                                               compiled as-is
                              ↓
platform/vita/src/
   ├── sys      threads, sync, input, display, memory
   ├── gpu      source GPU state → vitaGL / GXM
   ├── shader   vertex/fragment management, state-keyed generation
   ├── audio    software DSP emulator
   └── zeroed_new.cpp   operator new → zero-initialised memory
```

Note the last line: many console-era code bases assume `new` returns zeroed memory.
Replacing the global operator is cheaper than auditing the game.

## 3. Thread and priority mapping

Console priorities are not Vita priorities. Translate explicitly:

```
3DS: 2 app cores, priorities 0x18..0x3F
Vita: 3 app cores, priorities 64..191 (64 = highest)

vita_prio(p) = clamp(160 + (p - 0x30) * 2, 64, 191)
```

Core affinity is part of the design, not an afterthought:

| Work | Core |
|------|------|
| audio + decode | core 2 (highest priority) |
| render / job workers | core 1 |
| main / game logic | core 0 |

Affinity masks are `SCE_KERNEL_CPU_MASK_USER_0/1/2` (0x10000 / 0x20000 / 0x40000).

## 4. Lazy kernel objects

Console lock/event APIs allow zero-initialised (implicitly valid) objects. Vita
kernel objects need a UID. Bridge them with lazy creation so the game's statically
initialised locks keep working:

```c
/* state: 0 = uncreated, 1 = creating, 2 = ready */
[[nodiscard]] auto LazyUid(volatile int* state, int* uid, CreateFn create) noexcept -> int {
    if (*state == 2) return *uid;
    /* CAS 0 -> 1: the winner creates, everyone else spins briefly */
    ...
}
```

Without this, every global mutex in the game needs source changes — which defeats
the point of the route.

## 5. Shader generation from GPU state

Two working shapes, both proven:

| Shape | How | When |
|-------|-----|------|
| Runtime codegen | hash the GPU state (e.g. 64-byte fragment key: source registers, operators, functions per stage) → generate shader source → compile → **cache the binary on disk** | unbounded or unknown state space |
| Precompiled bank | N shaders + uber shader compiled offline with `psp2cgc`, selected by state key at runtime | fixed game, enumerable states |

mkwiita does the first (compiled programs cached under
`ux0:data/<app>/shaders/`). ACGC does the second (52 + 1 shaders, 3.4 MB generated
header). Both are validated by the same principle: **a state-keyed cache is what
makes the translation repeatable.**

## 6. Software DSP audio

Console audio hardware must become a mixer:

- decode: PCM8, PCM16, mono/stereo, ADPCM (8-byte frames, 14 samples each)
- resample: linear interpolation with fractional stepping
- per-voice IIR low-pass: `y = (1 - e) * x + e * y'`, `e = exp(-2 * pi * f0 / Fs)`
- mix: fold surround channels to front L/R
- run it at the highest priority on a dedicated core

This is the same shape as Melee's AX emulation (doc 10 §4) — hardware audio is
reproduced in software, and 24–64 voices is the achievable range.

## 7. Input: map to the host layout, not the source layout

| Source | Vita |
|--------|------|
| 3DS A | Cross |
| 3DS B | Circle |
| 3DS X | Triangle |
| 3DS Y | Square |

Keep confirm/cancel where Vita players expect them. Add a centre deadzone (±16
counts) before scaling stick axes.

## 8. Quality and debug switches as files

Shipping knobs as text files beats a settings UI you have not built yet:

```
ux0:data/<app>/quality.txt
  framerate=60
  resolution=1.5
  shadowVolumes=1
  bloomBlurRounds=2
```

Empty files as boolean switches (`gpu-sync.txt`, `no-shader-cache.txt`) let testers
disable a subsystem without a rebuild. It is crude and it works.

## 9. I/O tracing for free

Link-time wrapping:

```cmake
add_link_options(-Wl,--wrap=fopen -Wl,--wrap=fclose -Wl,--wrap=fread)
```

Count opens, measure duration, find the hot path. No source edits, no `#ifdef`s.
