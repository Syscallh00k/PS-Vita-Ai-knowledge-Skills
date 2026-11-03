---
name: decomp-port
description: Port a game you have source or a decompilation for — platform layer swap, ABI flags, subsystem order, big-endian data, release discipline.
when: A decompilation or source release exists for the target game.
---

# Skill: Decompiled Port

## Step 1 — Confirm the shape

| Requirement | Check |
|-------------|-------|
| Compilable source or matched decompilation | does it build for the original target first? |
| Data still user-supplied | you will stream from the user's own image, or convert offline |
| Platform APIs isolated | how much game code calls the console SDK directly? |

If the decompilation does not build for its original target, it will not build for
Vita either. Fix that first.

## Step 2 — One rule: do not edit game code

```
game code (unchanged)  →  platform layer (new)  →  Vita SDK
```

Everything platform-shaped goes in a new directory. The moment you patch game source
you lose the ability to merge upstream fixes.

## Step 3 — Flags that prevent silent corruption

```
-fno-short-enums        # 4-byte enums to match the console ABI
-fgnu89-inline
-fno-strict-aliasing
-fwrapv
-ffp-contract=off
-Wno-scalar-storage-order
```

`-fno-short-enums` alone prevents entire categories of "the game state is garbage".

## Step 4 — Link against stubs before writing a platform layer

Build the game code against empty stubs. You get a precise list of every function
you must implement, which is your work queue — far better than discovering them one
crash at a time.

**Gate:** it links. Not runs — links.

## Step 5 — Subsystem order

1. `main` runs and returns
2. display/VI brings up a clear colour
3. renderer draws one known scene
4. controller input
5. audio
6. saves
7. performance work

Do not start at 7. Every port that did burned out.

## Step 6 — Big-endian data

Two options:

| Route | Cost | Use when |
|-------|------|----------|
| GCC `scalar_storage_order("big-endian")` on structs | zero runtime, needs GCC ≥ 15, every struct must be right | you need zero-copy access to on-disc data |
| byte-swap on load into native structs | one pass, portable, easy to verify | you convert assets anyway |

Melee uses the attribute route because it streams from the user's ISO. A port that
converts data offline should swap on load — much easier to debug.

## Step 7 — Chunk the milestones

| Milestone | Definition of done |
|-----------|-------------------|
| M1 | boots to a colour |
| M2 | one static scene renders correctly |
| M3 | input works, camera controllable |
| M4 | audio, saves |
| M5 | 30 fps in the worst scene |
| M6 | menus, settings, packaging |

Each milestone should be demo-able. A port that cannot show M2 in the first month is
usually fighting the wrong problem.

## Step 8 — Release discipline

- one build script with `Release`/`Debug`
- Release = `-O3 -DNDEBUG`, no logging, no symbols, no debugger
- diagnostics behind explicit switches
- version in one file, consumed everywhere
- trained shader cache packaged into the VPK

## Step 9 — Diagnosis

| Symptom | Cause |
|---------|-------|
| structs full of noise | enum size or endianness |
| works until physics | float semantics — check `-ffp-contract`, `-fwrapv` |
| random crash after minutes | leaked memblocks or GPU memory exhaustion |
| everything runs at half speed | uncached writes to GPU-visible memory |
| audio desync | mixing on the render thread |

## Never

- never edit game source to fix a platform problem
- never start with the renderer before `main` and display work
- never compare performance without locking clocks first
