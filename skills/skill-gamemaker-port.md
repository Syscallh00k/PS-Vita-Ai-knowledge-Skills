---
name: gamemaker-port
description: Run a GameMaker Studio game on the Vita through YoYo Loader — softfp toolchain, asset swap for PC exports, memory reduction, keyboard mapping.
when: The target is a GameMaker title, or an Android binary that needs bionic pthread translation.
---

# Skill: GameMaker Port

## Step 1 — Determine the export type

| Export | Works |
|--------|-------|
| Android bytecode | yes, drag-and-drop |
| Android YYC (compiled) | yes |
| PC/console bytecode | yes, **after an asset swap** |
| PC/console YYC | no |

## Step 2 — Asset swap (PC exports only)

1. export a blank Android build from a GameMaker version **close to or equal to** the
   game's version
2. move the PC game's assets into that container
3. keep the runner/bytecode from the Android export — it is the part the loader
   executes

Version mismatch is the usual cause of "runs but everything is wrong".

## Step 3 — Build the softfp toolchain

The runner is softfp. Everything must be:

```
VitaSDK built with -mfloat-abi=softfp
openal-soft (vitaGL fork)  with -DCMAKE_C_FLAGS=-mfloat-abi=softfp
vitaGL                     make SOFTFP_ABI=1 NO_DEBUG=1 SHARED_RENDERTARGETS=1 HAVE_GLSL_SUPPORT=1
```

One hard-float library in the set produces ABI corruption that reads as random
memory bugs. There is no partial version of this rule.

## Step 4 — Install prerequisites on the console

| Requirement | Note |
|-------------|------|
| `kubridge.skprx` | mandatory |
| `fd_fix.skprx` | mandatory — **conflicts with rePatch**, tell users |
| `libshacccg.suprx` | shaders |

## Step 5 — Data layout

```
ux0:data/gms/<GAMENAME>/game.apk     ← user's own APK, renamed
```

## Step 6 — When it runs out of memory

In this order:

1. **externalise assets** with the loader's built-in function (moves them out of the
   APK)
2. **compress textures to PVRTC2** manually
3. optimise the APK (recompress entries, drop unread files) — also fixes streaming
   stutter
4. reduce audio quality

## Step 7 — Input

Two cases:

- game has controller code → map `SceCtrl` normally
- game expects a keyboard → map physical buttons to **synthetic key events**
  (the loader ships this; use it rather than patching the game)

## Step 8 — The transferable bits

Even if your target is not GameMaker:

| Pattern | Reuse |
|---------|-------|
| `*_soloader` pthread translation family | any Android binary that spawns threads — bionic and newlib structs differ, so each call is translated |
| `__android_log_print` / `scandir` / `__stack_chk_fail` shims | every Android loader |
| asset externalisation + PVRTC2 | any port with memory pressure |
| keyboard→button synthesis | PC-derived games with text-mode input |
| APK/archive optimiser | any port shipping a user-supplied container |

## Never

- never mix float ABIs
- never ship without documenting the `fd_fix` / rePatch conflict
- never assume a title that "runs" is playable — check input and audio separately
