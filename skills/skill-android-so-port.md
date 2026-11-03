---
name: android-so-port
description: Port an Android ARMv7 game by loading its libmain.so and reimplementing the OS around it — the soloader-template workflow end to end.
when: The target game is an Android build with armeabi-v7a native libraries.
---

# Skill: Android `.so` Port

## Step 0 — Is this the right route

| Check | Requirement |
|-------|-------------|
| Native libs are `armeabi-v7a` | Vita is ARMv7 — other ABIs are dead |
| Game logic is in the `.so` | not a thin Java/Unity-mono shell |
| Assets are readable files | not a single encrypted blob |
| GLES2-era renderer | vitaGL covers GLES2, not GLES3 features |

If any fails, stop and reconsider (docs 19, 26).

## Step 1 — Extract the game

`.apk` is a zip:

```
lib/armeabi-v7a/lib*.so     → ux0:data/<game>/
assets/                     → ux0:data/<game>/assets/
```

Copy the main `.so` and the assets. Note the exact filename — it goes into
`SO_PATH`.

## Step 2 — Toolchain

- VitaSDK built for **softfp** (`-mfloat-abi=softfp`), because the Android binary is
  softfp
- `kubridge.skprx` on the console (RWX loading)
- `libshacccg.suprx` if you use GLSL/GXP shaders

## Step 3 — Configure the template

```cmake
set(DATA_PATH "ux0:data/gamename/")
set(SO_PATH   "${DATA_PATH}libmain.so")
add_definitions(-DDATA_PATH="${DATA_PATH}" -DSO_PATH="${SO_PATH}")
option(NDK_PORT "game has android_main" OFF)
option(USE_SCELIBC_IO "SceLibcBridge for IO" ON)
set(SHADER_FORMAT "GLSL")            # GLSL | CG | GXP
```

## Step 4 — Boot order (do not reorder)

```
clocks → fios_init → kubridge check → so exists → so_file_load →
settings_load → so_relocate → resolve_imports → so_patch →
so_flush_caches → so_initialize → gl_preload → jni_init → controls_init
```

Then call the entry point:

- JNI game: `JNI_OnLoad(&jvm)` via `so_symbol`
- NDK game: build a fake `ANativeActivity`, call `ANativeActivity_onCreate`, then
  drive `onStart`, `onResume`, `onInputQueueCreated`, `onNativeWindowCreated`

## Step 5 — Grow `dynlib.c`

Every unresolved symbol is a crash waiting. Workflow:

1. run, read the log, find the first unresolved import
2. either implement it, or map it to an existing Vita function
3. repeat

`so_resolve_with_dummy()` lets you continue past a symbol you do not understand yet —
useful for discovering the *next* missing thing, never for shipping.

## Step 6 — Patch the platform

Common patches, in the order they usually appear:

| Patch | Why |
|-------|-----|
| `kuser_cmpxchg` / `kuser_memory_barrier` at `0x9A000FA0`/`0x9A000FC0` | Android calls fixed-address helpers Vita does not have |
| path redirects (FIOS overlay) | the game hardcodes `/data/...` |
| `so_symbol_fix_ldmia` | unaligned `LDMIA` raises SIGBUS on some setups |
| input mapping | `SceCtrl` → Android key/touch/analog events |
| shader interception | `glShaderSource`/`glCompileShader` hooks |

## Step 7 — Shader strategy

| Format | Setup | Use when |
|--------|-------|----------|
| GLSL | pass through to vitaGL at runtime | fastest to try |
| CG | hand-translate, load `.cg` by hash | vitaGL translation fails on some shaders |
| GXP | precompile with `psp2cgc`, ship `.gxp` | production, no runtime compile |

Turn on `DUMP_COMPILED_SHADERS` once shaders work, so later launches skip compilation.

## Step 8 — Diagnosis

| Symptom | Cause |
|---------|-------|
| crash in `JNI_OnLoad` | FalsoJNI returning wrong types |
| black screen, alive | GL never initialised — `gl_preload()` before game GL calls |
| files not found | overlay rules missing |
| crash right after load | missing cache flush, or unrelocated import |
| random SIGBUS | unaligned access — apply the ldmia fix |
| looks wrong, no crash | unsupported GLES feature — disable that effect |

## Step 9 — Ship

Data on `ux0:`, loader in the VPK, boot gates that print a clear message when the
`.so` or assets are missing. Users will install data wrong; make the error tell them
the exact path it expected.

## Never

- never ship with `so_resolve_with_dummy` still stubbing a symbol the game needs
- never assume the game's `.so` is the only one — check for extra `.so` files in
  `lib/armeabi-v7a`
- never redistribute the game's `.so` or assets
