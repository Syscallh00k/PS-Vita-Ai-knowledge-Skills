# 19 — Porting Android `.so` Games

The cheapest real ports: the game binary already runs on ARMv7. You reimplement the
OS around it.

Source material: `soloader-template`, `Other Ports/raider-vita-master`.

---

## 1. Why this works

| Fact | Consequence |
|------|-------------|
| Vita CPU is ARMv7 | Android `armeabi-v7a` code executes natively |
| Android binaries are position-independent ELF | loadable at a chosen address, relocatable |
| Most game logic is in the `.so`, not Java | a fake JVM only needs to satisfy init |
| Graphics go through GLES2 | `vitaGL` translates GLES2 → GXM |

```
 ┌──────────────────────── Vita ─────────────────────────┐
 │ loader: libc_bridge · fios · FalsoJNI · vitaGL       │
 │ ┌──────────────────────────────────────────────────┐ │
 │ │ libmain.so — original game code, byte-exact      │ │
 │ └──────────────────────────────────────────────────┘ │
 └──────────────────────────────────────────────────────┘
```

## 2. Prerequisites

| Item | Why |
|------|-----|
| VitaSDK built with **softfp** | the Android binary's float ABI is softfp, not hard |
| `kubridge.skprx` | RWX memory for loading and patching |
| `libshacccg.suprx` | shader compilation if you use GLSL/GXP |
| The game's `.apk` (your own purchase) | source of `lib/armeabi-v7a/*.so` and `assets/` |

## 3. Project configuration

`soloader-template/CMakeLists.txt` exposes the decisions that matter:

```cmake
set(DATA_PATH "ux0:data/gamename/")
set(SO_PATH   "${DATA_PATH}main.so")
add_definitions(-DDATA_PATH="${DATA_PATH}" -DSO_PATH="${SO_PATH}")

option(NDK_PORT "Android NDK game with android_main" OFF)
option(USE_SCELIBC_IO "Use SceLibcBridge for IO" ON)
set(SHADER_FORMAT "GLSL")   # GLSL | CG | GXP
option(DUMP_COMPILED_SHADERS "Cache compiled shaders on disk" OFF)
```

- `NDK_PORT` changes the entry path from `JNI_OnLoad` to
  `ANativeActivity_onCreate` + lifecycle callbacks
- `USE_SCELIBC_IO` pulls in FIOS path overlays — usually required
- `SHADER_FORMAT` decides whether shaders are translated, hand-converted, or
  precompiled

## 4. Boot sequence (`soloader_init_all`)

```
 1. parse -config launch param → optionally exec app0:/configurator.bin
 2. lock clocks 444 / 222 / 222 / 166
 3. fios_init(DATA_PATH)
 4. assert kubridge loaded
 5. assert the .so exists
 6. so_file_load(&so_mod, SO_PATH, 0x98000000)
 7. settings_load()
 8. so_relocate()
 9. resolve_imports()      ← dynlib.c, the big table
10. so_patch()             ← kuser atomics + your hooks
11. so_flush_caches()
12. so_initialize()        ← init_array constructors
13. gl_preload()
14. jni_init()
15. controls_init()
```

Load address `0x98000000` is the template's choice; any free region works, but keep
it fixed so `noASLR` and hardcoded addresses stay meaningful.

## 5. The reimplementation surface

| Shim | Directory | Responsibility |
|------|-----------|----------------|
| `libc_bridge` | `lib/libc_bridge/` | make bionic-ish `fopen`/`stat`/`pthread` calls land on Vita libc |
| `fios` | `lib/fios/` | path overlays so `/data/...` resolves into `ux0:data/game/` |
| `FalsoJNI` | submodule | fake JVM: `JNI_OnLoad`, `FindClass`, `CallStatic*` returning plausible values |
| `FalsoNDK` | submodule | fake `ANativeActivity`, input queue, native window |
| `controls` | `source/reimpl/controls.c` | map `SceCtrl` to Android key/touch/analog events |
| `time64` | `source/reimpl/time64.c` | 32-bit `time_t` conversion |
| `egl`/`glutil` | `source/reimpl/egl.c`, `utils/glutil.c` | EGL + GLES2 onto vitaGL |

## 6. Patching: `kuser` helpers

Android binaries call `__kuser_cmpxchg` / `__kuser_memory_barrier` at fixed
addresses (`0xFFFF0FC0`, `0xFFFF0FA0`). Vita has no such page, so the loader creates
one and rewrites the call sites:

```c
/* allocate a page at 0x9A000000, make it RWX, place trampolines */
hook_addr(0x9A000FA0, (uintptr_t) __kuser_memory_barrier);
hook_addr(0x9A000FC0, (uintptr_t) __atomic_cmpxchg);

/* patch the binary's literal references to the old addresses */
for (uint32_t addr = so_mod.text_base; addr < so_mod.text_base + so_mod.text_size; addr += 4) {
    uint32_t* a = (uint32_t*) addr;
    if (*a == 0xFFFF0FC0) { uint32_t p = 0x9A000FC0; kuKernelCpuUnrestrictedMemcpy(a, &p, 4); }
    if (*a == 0xFFFF0FA0) { uint32_t p = 0x9A000FA0; kuKernelCpuUnrestrictedMemcpy(a, &p, 4); }
}
```

Scanning the whole text segment for those constants is the standard trick.

## 7. The realistic failure modes

| Failure | Cause | Fix |
|---------|-------|-----|
| crashes in `JNI_OnLoad` | FalsoJNI returns wrong types | log every call, return conservative values |
| black screen, no crash | GLES never initialised | `gl_preload()` before any game GL call |
| files not found | paths still Android-absolute | add FIOS overlay rules |
| random crashes | missing cache flush after patching | `so_flush_caches` |
| SIGBUS on unaligned access | ARMv7 without fault trapping | `so_symbol_fix_ldmia` |
| glitchy rendering | GLES feature vitaGL does not implement | disable that effect |

## 8. What this teaches even for non-Android targets

The **shape** — load foreign code, resolve its imports to your shims, patch the
mismatches — is the same shape used for PPU static recompilation, for running old
engine modules, and for any "the binary is fixed, the OS is ours" port.

## 9. Sources

- `soloader-template/` (CMakeLists, `source/utils/init.c`, `source/patch.c`,
  `source/reimpl/*`, `lib/fios`, `lib/libc_bridge`)
- `Other Ports/raider-vita-master/loader/so_util.c`, `main.c`, `loader/elf.h`
