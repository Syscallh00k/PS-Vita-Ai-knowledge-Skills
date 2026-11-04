# 52 — Building Existing Projects on Vita (without a rewrite)

How to make someone else's Makefile or CMake tree produce a VPK, with the patterns
that actually appear in shipping ports.

> Sources: `new imports/fake-08-master/platform/vita/Makefile`,
> `new imports/flare-engine-master/Makefile.vita`,
> `new imports/stepmania-psvita-main/Build-Vita/`,
> `new imports/Aleph-One-PsVita-main/CMakeLists.txt`,
> `new imports/DeadBabiesX-psvita-port-main`. See [CREDITS](../CREDITS.md).

---

## 1. The three integration shapes

| Shape | When | Example |
|-------|------|---------|
| **Platform Makefile** (`Makefile.vita`) | project already has per-platform makefiles | FLARE, fake-08 |
| **CMake toolchain + cache preset** | project uses CMake, options galore | StepMania, Aleph One |
| **Upstream build system + Vita target** | project has SCons/autotools with host detection | Godot 4, CPython |

Pick whichever costs less diff. Never rewrite the build system to "make it clean" —
that is how a port becomes unportable.

## 2. Pattern A — `Makefile.vita`

fake-08's Vita makefile is ~40 lines and is the whole integration:

```make
PROJECT_TITLE   := FAKE-08
PROJECT_TITLEID := FAKE00008

SOURCES  := ${SOURCES} ../SDL2Common/source source
LIBS = -lSDL2 -lScePower_stub -lSceDisplay_stub -lSceCtrl_stub -lSceAudio_stub \
       -lSceSysmodule_stub -lSceGxm_stub -lSceCommonDialog_stub -lSceAppMgr_stub \
       -lSceTouch_stub -lSceHid_stub -lSceMotion_stub -lm

CFLAGS = -g -Wl,-q -O2 -ftree-vectorize -MMD -MP -D__VITA__ $(INCLUDE)

$(PROJECT).vpk: eboot.bin param.sfo
	vita-pack-vpk -s param.sfo -b eboot.bin \
		--add sce_sys/icon0.png=sce_sys/icon0.png \
		--add sce_sys/livearea/contents/bg.png=sce_sys/livearea/contents/bg.png \
		$(PROJECT).vpk

$(PROJECT).velf: $(PROJECT).elf
	$(STRIP) -g $<
	vita-elf-create $< $@

vpksend: $(PROJECT).vpk
	curl -T $(PROJECT).vpk ftp://$(PSVITAIP):1337/ux0:/
```

Note the trick: **reuse the shared platform source directory** (`../SDL2Common`) so
the Vita build shares SDL glue with other consoles. If the project has a
`platform/` folder with other targets in it, you are adding a sibling, not a fork.

## 3. Pattern B — CMake via `-C` initial cache

StepMania 5 needed ~20 option overrides *before* its own CMake logic ran. The solution
is an initial-cache file passed with `-C`:

```sh
cmake -S stepmania-5.0.12 -B build \
  --toolchain "$VITASDK/share/vita.toolchain.cmake" \
  -C Build-Vita/vita-preset.cmake \
  -DVITA=ON -G Ninja
```

```cmake
set(VITA ON CACHE BOOL "" FORCE)
set(WITH_GLES2 ON CACHE BOOL "vitaGL" FORCE)
set(WITH_NETWORKING OFF CACHE BOOL "" FORCE)
set(WITH_FFMPEG OFF CACHE BOOL "no video decode on stage 1" FORCE)
set(WITH_SSE2 OFF CACHE BOOL "N/A on ARM" FORCE)
set(WITH_LTO OFF CACHE BOOL "" FORCE)
set(WITH_PORTABLE_TOMCRYPT ON CACHE BOOL "no asm" FORCE)
add_compile_definitions(VITA __vita__ HAVE_GLES2 WITH_GLES2)
```

Why `-C` and not `-D`: `-D` lands after the project's own `set(... CACHE ...)`
defaults in some trees, and you spend an hour discovering that your flag was ignored.
`-C` is guaranteed to be in the cache first.

Aleph One shows the minimal CMake version:

```cmake
include("$ENV{VITASDK}/share/vita.cmake" REQUIRED)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D__vita__ -Wno-error")
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -D__vita__")
# sources + Source_Files/vita_init.c
vita_create_self(eboot.bin alephone UNSAFE)
vita_create_vpk(alephone.vpk ALPH00001 eboot.bin ...)
```

`-Wno-error` matters: old engines do not survive modern GCC's warning set, and a
port is not the place to fix 400 warnings.

## 4. Turn off what you cannot support, loudly

Every successful port disables features rather than half-implementing them, and says
so:

| Disabled | Port | Stated reason |
|----------|------|---------------|
| FFmpeg / video | StepMania | "no video playback on stage 1" |
| networking | StepMania stage 1 | online play is a later stage |
| SSE2 | StepMania | N/A on ARM |
| LTO | StepMania, FLARE | crashes / not worth it |
| asm in crypto | StepMania | "portable tomcrypt" |

Write the list in the README. A feature that silently does nothing is a bug report
you will receive forever.

## 5. The flag set that appears everywhere

```
-g -Wl,-q -O2 [-Ofast] -ffast-math -fno-strict-aliasing
-fno-exceptions -fno-optimize-sibling-calls -fno-lto
-std=gnu++11 / gnu++17
-D__vita__ [-DVITA] [-D__VITA__]
```

and the link side:

```
-Wl,--whole-archive pthread -Wl,--no-whole-archive
stdc++ m c
-lSceKernelDmacMgr_stub     (vitaGL)
-lSceShaccCg_stub -lSceShaccCgExt -lvitashark   (runtime shaders)
```

## 6. Packaging: three ways, all fine

| Tool | Line |
|------|------|
| `vita_create_vpk` (CMake) | declarative, reproducible |
| `vita-pack-vpk` (Make) | explicit file list |
| `7z a -tzip` | what melonDS does; works, but you must add `sce_sys` correctly |

Whichever you pick, the VPK layout rules from doc 03 still apply.

## 7. Keep the unstripped ELF

Both melonDS and the Godot packager do this, and both are right:

```make
%.velf: %.elf
	cp $< $<.unstripped.elf
	$(PREFIX)-strip -g $<
	vita-elf-create $< $@
```

Packet it with the release. A build without symbols is a build you cannot debug later.

## 8. Integration checklist

- [ ] did I add a platform target instead of editing existing ones?
- [ ] are build-system options overridden early (`-C`, not `-D`)?
- [ ] is every unsupported feature disabled **and documented**?
- [ ] `-Wl,-q` present, pthread whole-archived, stdc++ linked?
- [ ] unstripped ELF archived per build?
- [ ] does the Makefile/CMake produce the VPK itself, or is packaging a second step
      someone will forget?
- [ ] is there an FTP `send`/`vpksend` target for iteration?
