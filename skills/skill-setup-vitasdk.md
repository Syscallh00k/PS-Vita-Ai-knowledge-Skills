---
name: setup-vitasdk
description: Install and verify a working VitaSDK, then add the third-party libraries a real port needs, in dependency order.
when: First time on a machine, or a build says the toolchain, vita.cmake, or a library is missing.
---

# Skill: Setup VitaSDK

## Step 1 — Does the SDK actually work

```powershell
$env:VITASDK = 'C:\vitasdk'
$env:PATH = "$env:VITASDK\bin;$env:PATH"

arm-vita-eabi-gcc --version
Test-Path $env:VITASDK\share\vita.cmake
Test-Path $env:VITASDK\share\vita.toolchain.cmake
Test-Path $env:VITASDK\arm-vita-eabi\lib\libSceGxm_stub.a
```

All four must succeed.

**Gate.** If `share/` or `bin/` is missing you have a headers-and-stubs extract,
not an SDK. Use the official installer (Windows/Linux/macOS) or build from
`vitasdk/buildscripts`. Do not hand-assemble a toolchain — the GCC build needs
Vita-specific patches.

## Step 2 — Confirm the version

```powershell
arm-vita-eabi-gcc -dumpversion
```

Note it. GCC **15 or newer** is required for big-endian `scalar_storage_order`
(needed by console-port code that mirrors on-disc structs).

## Step 3 — Third-party libraries, in this order

| # | Library | Depends on | Needed for |
|---|---------|-----------|-----------|
| 1 | zlib, libpng, jpeg/turbojpeg | — | asset decode |
| 2 | vita2d | GXM stubs | 2D, textures, simplest first frame |
| 3 | SceShaccCgExt | — | RE'd extended shader-compiler headers |
| 4 | vitaShaRK | 3 | runtime Cg compilation |
| 5 | kubridge | — | RWX patching (only for loaders/hookers) |
| 6 | libmathneon | — | NEON math helpers |
| 7 | vitaGL | 2/3/4 | GL/GLE2 path (only if needed) |
| 8 | imgui, SDL2 (vitaGL builds) | 7 | debug UI, portability layer |

Rules:

- install 3 **before** 4, always
- do not mix releases of 3 and 4 — a version skew produces shaders that compile and
  render garbage
- each is `make install` into `$VITASDK/arm-vita-eabi`

## Step 4 — Prove it with a build

```powershell
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j8
```

Expect an `.vpk`. If CMake complains about the toolchain, `VITASDK` is not visible
to the CMake process — set it in the same shell.

## Step 5 — Prove it on the console

Install the VPK with VitaShell, or push and launch over FTP:

```powershell
curl -T build/eboot.bin ftp://$env:PSVITAIP:1337/ux0:/app/XXXX00001/eboot.bin
echo "launch XXXX00001" | nc $env:PSVITAIP 1338
```

**Gate.** Your own bubble boots. Until this passes, nothing else you learn is real.

## Step 6 — Console prerequisites

| Requirement | Why |
|-------------|-----|
| HENkaku/taiHEN + "Enable Unsafe Homebrew" | unsigned code |
| `ur0:data/libshacccg.suprx` | runtime shader compiler |
| `kubridge.skprx` | only if you patch code pages |
| `fd_fix.skprx` | only for Android-derived ports |
| PSVshell (optional) | 500 MHz CPU / 222 MHz GPU |

## Never

- never edit files inside `$VITASDK` to fix a build error — fix the build
- never assume the user has your plugins; check them at boot and fail loudly
