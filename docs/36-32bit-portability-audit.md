# 36 — 32-bit & Portability Audit

What breaks when a 64-bit or 32-bit-x86 codebase meets an ARM Cortex-A9, and the one
trap that survives compilers and reviewers alike.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/runtime.md` §10, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Type width

- `long`, `size_t`, pointers are **32-bit**. Replace `long` assumptions with
  `int64_t` / `intptr_t`; never use `long` to hold a pointer or a file size.
- **plain `char` is unsigned on ARM.** Code written for x86 needs `-fsigned-char`
  (Quake, Xash3D-era engines).
- enums are 1 byte by default in the ARM EABI. See §4.

## 2. Alignment

Unaligned loads from packed file structs can fault. Two options:

| Option | Cost |
|--------|------|
| `-fmax-type-align=1` for that translation unit | slower, safe |
| `memcpy` into a local before use | slower where used, correct |

Start with `memcpy`; only relax after profiling proves it matters.

## 3. Missing syscalls

| Missing | Workaround |
|---------|-----------|
| `fork` / `exec` | not available — restructure |
| `dlopen` | `vita-libdl`, or static linking (`XASH_STATIC`) |
| file-backed `mmap` | read the file |
| anonymous `mmap` / `mprotect` | emulate on kubridge (doc 37) |
| `std::filesystem`, `getenv`-driven config | replace with fixed `ux0:data` paths |

## 4. The enum split — read this twice

`-fno-short-enums` is mandatory for console-derived code, and it **splits the build**:

- arm-vita-eabi defaults to short (1-byte) enums
- every prebuilt SDK library (newlib, libstdc++, SDL) uses the default
- the guest/console data structures assume **int-sized** enums, because that is what
  the console had

So: compile game code with `-fno-short-enums`, keep the default for files that
include SDK headers, and **never let an enum-typed value cross that boundary** — pass
plain integers.

## 5. Optimisation flags

```
-O2 / -O3 -ffast-math      usual choice
-flto                      treat as risky: at least one port enabled it and later
                           removed it because it "causes crashes" — A/B on hardware
-Wl,--wrap=memcpy,...      wrap to sceClibMem* only when profiling shows it matters
```

## 6. The trap: console hardware checks still fire

Console code branches on **physical addresses** that are only meaningful on the
original hardware: arena bounds, MEM1/MEM2 ranges, `0x80000000`-based pointer
validation, `OSPhysicalToCached` round-trips, alignment asserts on `0x8xxxxxxx`
pointers.

On a Vita process the pointers are relocated — the executable, the newlib heap and
every memblock live outside the guest arena — so those comparisons *should* be
false. They are not, because the interesting thresholds fall inside the Vita's own
address space.

Measured case (`mDoMch_Create()` in a Twilight Princess port):

```cpp
// retail-only trim: "reclaim the 24 MiB the dev hardware reserved"
if (arenaHi > 0x81800000 && arenaHi - 0x1800000 > arenaLo)
    OSSetArenaHi((void*)(arenaHi - 0x1800000));
```

`arenaHi` was `0x871481a8`, so the branch fired **every run** and silently deleted
24 MiB of arena:

| | arena hi | size |
|---|---|---|
| trim live | `0x859481c0` | 40 MiB |
| trim compiled out | `0x871481a8` | 64 MiB |

The symptom was not a clean OOM — it was an intermittent corrupted allocation size,
roughly one run in seven.

**Rule:** for any branch comparing a pointer or size against a fixed hardware
constant, do not reason about it — measure it. Log both operands before and after,
then rerun with the block compiled out and diff. The tell was `0x1800000` appearing
exactly in the delta. Guard with `#if !TARGET_VITA` and put the measurement in a
comment, or the next merge re-adds it.

## 7. Companion trap: `TARGET_PC`-style macros

Ports usually define `TARGET_PC`/`MELEE_PC` so upstream code compiles. That means
desktop-only arithmetic runs unguarded on the Vita unless you add a platform guard.
A factor described as "×20 to fix desktop allocation failures" is exactly where to
look.

## 8. Audit tooling

`audit_source.py` (upstream, see doc 39) greps a tree for these categories and prints
a feasibility hint. It is a grep, not a compiler — expect false positives (comments,
`#ifdef`-ed out code) and read the hits.
