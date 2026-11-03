# 02 — VitaSDK Reference

What the SDK tree on this machine actually contains, what each module area is for,
and what you have to add yourself.

---

## 1. What is present

```
Sdk/
├── include/
│   ├── vitasdk.h  vitasdkkern.h      umbrella headers
│   ├── psp2/                         79 module headers (userland)
│   ├── psp2common/                   shared types
│   ├── psp2kern/                     kernel headers (plugins)
│   ├── psp2kern/…                    ksce* APIs
│   └── c++/15.2.0/                   libstdc++ headers
└── lib/
    ├── crt0.o  libc.a  libm.a  libpthread.a  libg.a
    ├── libSce*_stub.a                link stubs for every sysmodule
    ├── libSce*_stub_weak.a           weak variants
    └── liblibScePiglet_stub.a        GLES backend
```

**GCC 15.2.0** is the toolchain version implied by the C++ header path — relevant
because Melee's port needs GCC ≥ 15 for big-endian `scalar_storage_order`.

## 2. What is missing

| Missing | Why it matters |
|---------|----------------|
| `share/vita.cmake` | the CMake macros (`vita_create_self`, `vita_create_vpk`) live here — no CMake build without it |
| `bin/` | `arm-vita-eabi-gcc`, `vita-make-fself`, `vita-mksfoex`, `vita-pack-vpk`, `psp2cgc` |
| `share/vita.toolchain.cmake` | toolchain file referenced by every project |
| `vita2d`, `vitaGL`, `vitaShaRK`, `SceShaccCgExt`, `kubridge`, `libmathneon`, `imgui`, SDL2 | third-party, installed separately (skill: setup-vitasdk) |

This tree is a **headers + stubs extract**. It is useful for reading API truth
(what `SceGxm` really exposes, what constants exist) but not for building.

## 3. The module headers worth knowing

All 79 live in `Sdk/include/psp2/`. The ones that matter for ports:

| Header | API | Use |
|--------|-----|-----|
| `gxm.h` (115 KB — by far the largest) | `sceGxm*` | rendering |
| `shacccg.h` | `sceShaccCg*` | runtime shader compile |
| `display.h` | `sceDisplay*` | framebuffer, vblank |
| `ctrl.h` | `sceCtrl*` | buttons, sticks |
| `touch.h` | `sceTouch*` | front/rear touch |
| `motion.h` | `sceMotion*` | accelerometer/gyro |
| `audioout.h` / `audioin.h` | `sceAudioOut*` / `sceAudioIn*` | PCM out/in |
| `atrac.h` | `sceAtrac*` | ATRAC9 hardware decode |
| `avplayer.h` (10 KB) | `sceAvPlayer*` | H.264 playback |
| `videodec.h` / `audiodec.h` | `sceVideoDec*` | codec-level access |
| `fios2.h` (via `fios2kernel.h`) | `sceFios*` | async file I/O |
| `io/fcntl.h`, `io/stat.h`, `io/dirent.h` | `sceIo*` | sync file I/O |
| `power.h` | `scePower*` | clock control |
| `appmgr.h` (18 KB) | `sceAppMgr*` | launch params, relaunch |
| `apputil.h` | `sceAppUtil*` | app events, save data |
| `npdrm.h` | `sceNpDrm*` | license queries |
| `paf.h`, `shellutil.h`, `promoterutil.h` | `scePaf*` | LiveArea/promoter hacks |
| `common_dialog.h` | `sceCommonDialog*` | system dialogs |
| `sysmodule.h` | `sceSysmodule*` | module loading |
| `jpeg.h` / `jpegarm.h` | hardware JPEG decode | fast texture decode |
| `pgf.h` / `pvf.h` | font rendering | text |
| `rtc.h` | `sceRtc*` | wall clock |
| `registrymgr.h` | `sceRegistry*` | settings |
| `imc`/`appmgr` module loading rules | — | where files may live |

Kernel-side equivalents live under `Sdk/include/psp2kern/` (`ksce*`), which is
what `skprx` plugins use.

## 4. Linking

```cmake
target_link_libraries(app
    SceDisplay_stub SceGxm_stub SceCtrl_stub SceAudio_stub
    SceFios2_stub SceAppMgr_stub SceSysmodule_stub ScePower_stub
    -Wl,--whole-archive pthread -Wl,--no-whole-archive
    m stdc++)
```

- Every sysmodule is a **stub** library: it resolves NIDs at load time against the
  firmware. You cannot call a module that is not loaded (`sceSysmoduleLoadModule`).
- `--whole-archive pthread` is required or the thread glue gets dropped.
- `libSceShaccCg_stub.a` is present — proof that runtime Cg compilation is a
  first-class SDK feature, not a hack.

## 5. Reading the headers is the fastest way to settle an argument

Two examples that have saved real debugging time:

- `gxm.h` contains the full `SceGxmTextureFormat` enum — the only authoritative
  list of what the GPU can sample.
- `avplayer.h` documents the `SceAvPlayerFrameInfo` layout, which tells you
  whether a video path can be zero-copy.

Rule: if two people disagree about a Vita limit, grep `Sdk/include/psp2`.
