# 43 — GameMaker Games: YoYo Loader

The runtime route for GameMaker Studio titles: load the official Android runner
(`libyoyo.so`), reimplement Android around it, and run the game's own bytecode.

> Source: `Other Emulators/yoyoloader_vita-master` (Rinnegatamante).
> See [CREDITS](../CREDITS.md).

---

## 1. What it loads and what it does not

| Game export type | Status |
|------------------|--------|
| Android bytecode | native |
| Android YYC (compiled) | native |
| PC/console bytecode | works **after an asset swap** |
| PC/console YYC | no |

"Asset swap" means: take a blank Android export from a matching GameMaker version and
move the PC game's assets into it. The runner is what handles the bytecode; you are
only supplying a container it accepts.

## 2. Why this exists as its own route

GameMaker games are not ARM binaries you can port source-wise, and not Unity/UE
titles. The runner interprets the game's own bytecode, so:

- game logic needs no reimplementation
- compatibility depends on the runner version, not on the game
- the loader owns input, assets, audio, and packaging

It is the same family as [doc 19](19-porting-android-so.md) (load a foreign ARM
binary, reimplement the OS) with a fixed, known payload instead of an arbitrary game.

## 3. Build requirements — softfp everywhere

The Android runner is softfp, so the whole toolchain must be:

```
VitaSDK built with -mfloat-abi=softfp
ALL libraries rebuilt with -mfloat-abi=softfp
```

Libraries needed, in practice:

| Library | Build note |
|---------|-----------|
| `openal-soft` (vitaGL fork) | `cmake -DCMAKE_C_FLAGS=-mfloat-abi=softfp` |
| `vitaGL` | `make SOFTFP_ABI=1 NO_DEBUG=1 SHARED_RENDERTARGETS=1 HAVE_GLSL_SUPPORT=1 install` |
| the rest of the usual set | prebuilt softfp packages exist |

Mixing one hard-float library into a softfp build produces ABI mismatches that look
like random memory corruption. There is no partial version of this rule.

## 4. What the loader reimplements

From `loader/main.c` (97 KB) — the surface a GameMaker runner needs:

| Area | Functions |
|------|-----------|
| pthreads | a full `*_soloader` family: create/join/detach, mutex init/trylock/unlock, cond wait/timedwait/broadcast, attr set/get stack size and sched param, `pthread_getattr_np`, `pthread_equal` |
| Android logging | `__android_log_print`, `__android_log_vprint` |
| Directory scan | `scandir_hook` returning `android_dirent` |
| Stack guard | `__stack_chk_fail_fake` |
| Input | `patch_gamepad`, `GamePadUpdate` |
| Config | `loadConfig(game)` per title |

**The `*_soloader` pthread family is the interesting artefact.** Android's bionic
pthreads and newlib's pthreads do not share struct layouts, so every call is
translated rather than forwarded. If you ever port an Android binary that creates
threads, expect to write this layer; it is dozens of functions, not one.

## 5. Asset handling — where the memory goes

| Feature | Why |
|---------|-----|
| built-in APK optimiser (Triangle in the selector) | recompresses entries so streaming does not stutter, removes files the runner never reads |
| asset externalisation | moves assets out of the APK to `ux0:data`, freeing resident memory |
| PVRTC2 texture compression | the single biggest memory win for texture-heavy titles |

If a game runs out of memory, that is the order to try things:
externalise → PVRTC2 → optimise the APK → reduce audio quality.

## 6. Input

Two paths:

1. the game has working controller code → map `SceCtrl` normally
2. the game expects keyboard → **map physical buttons to keyboard keys** (a shipped
   feature, with a keymap format and per-game config)

That second path is the general lesson: for any PC-derived game, "the game reads a
keyboard" is a solvable problem without touching the game — synthesise the key
events.

## 7. Packaging and user contract

```
ux0:data/gms/<GAMENAME>/game.apk     ← the user's own APK, renamed
```

- `kubridge.skprx` + `fd_fix.skprx` required (and `fd_fix` conflicts with rePatch —
  say so)
- `libshacccg.suprx` required for shaders
- the loader ships: a game selector with banners, translations, per-game config

Note the shape: the loader is a **platform**, not a single port. Compatibility is
tracked per game in a public list, exactly like an emulator.

## 8. Could this apply to our targets?

Only if the target is a GameMaker title. The transferable parts:

| Pattern | Reusable for |
|---------|-------------|
| `*_soloader` pthread translation layer | any Android binary that spawns threads |
| asset externalisation + PVRTC2 | any port with memory pressure |
| keyboard-to-button mapping | any PC-derived game whose input is text-mode |
| APK/archive optimiser | any port that ships a user-supplied container |
