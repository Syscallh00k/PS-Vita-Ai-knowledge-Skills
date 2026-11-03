# 44 — Emulator Port Anatomy

What a Vita emulator port actually consists of, using the three emulators in this
workspace as the map: **DaedalusX64** (N64, dynarec + vitaGL), **melonDS** (DS,
interpreter + vita2d), **PCSX AwO** (PS1, dynarec + vitaGL).

> Sources: `Other Ports/DaedalusX64-vitaGL-master`, `Other Ports/melonDS-Vita-master`,
> `Other Emulators/PCSX_AwO-master`. See [CREDITS](../CREDITS.md).

---

## 1. The platform directory is the port

Everything else is the emulator's own code. DaedalusX64 says it plainly:

```
Source/
  Core/  DynaRec/  Graphics/  HLEAudio/  HLEGraphics/  Input/  Interface/  Math/
  OSHLE/  Plugins/  System/  Utility/          ← upstream emulator
  SysVita/                                     ← the port
```

`SysVita/` contains:

| Area | Files |
|------|-------|
| dynarec backend | `DynaRec/CodeGeneratorARM.cpp` (97 KB), `AssemblyWriterARM.cpp`, `CodeBufferManagerARM.cpp`, `DynarecStubARM.S`, `N64RegisterCacheARM.cpp` |
| graphics | `GraphicsContextVita.cpp`, `NativeTextureVita.cpp` |
| HLE graphics | `GraphicsPluginVita.cpp`, `RendererLegacy.cpp`, `RendererModern.cpp`, `Accurate/RendererAccurate.cpp`, `Combiner/*` |
| HLE audio | `AudioPluginVita.cpp`, `AudioOutput.cpp` |
| input | `InputManagerVita.cpp` (26 KB) |
| UI | `MainMenuScreen.cpp`, `MenuBarScreen.cpp`, `InGameMenuScreen.cpp`, `DownloaderScreen.cpp` |
| utility | `IOVita.cpp`, `ThreadVita.cpp`, `TimingVita.cpp` |
| entry | `main.cpp` |

**Reading order for any emulator port:** `main.cpp` → input → timing → graphics
context → audio → menus. That is also the order in which things start working.

## 2. Two renderer strategies

| Emulator | Guest GPU | Approach |
|----------|-----------|----------|
| melonDS | DS 2D/3D | software rasterizers (`GPU2D`, `GPU3D_Soft`) blitted through `vita2d` — no shaders at all |
| PCSX AwO / DaedalusX64 | PS1 GS / N64 RDP | HLE → vitaGL draws, with per-state shaders |

The melonDS route is underrated: if the guest GPU is simple enough, a software
rasteriser plus a blit can be less work and *more* correct than a shader pipeline,
and it sidesteps `libshacccg` entirely.

## 3. Dynarec vs interpreter

| | Dynarec | Interpreter |
|---|---|---|
| Speed | required for N64/PS1 at speed | fine for DS-era ARM at moderate clocks |
| Complexity | a register cache, an ARM code generator, cache flush discipline | straightforward |
| Memory | needs executable memory (VM block or kubridge) | none |
| Port surface | `CodeGenerator*`, `AssemblyWriter*`, `DynarecStub*.S` | nothing |

DaedalusX64's Vita dynarec lives entirely in `SysVita/DynaRec` and reaches the guest
registers through a hand-written `N64RegisterCacheARM`. If you are porting an
emulator that already has a dynarec for ARM (PSP ports are usually ARM), that
subdirectory is your work.

## 4. vitaGL build flags are part of the port

DaedalusX64 states its required vitaGL configuration explicitly:

```
READBACKS_SPEEDHACK=1 HAVE_GLSL_TEXTURE_SIZE=1 NO_DEBUG=1 HAVE_UNFLIPPED_FBOS=1
NO_TEX_COMBINER=1 HAVE_CUSTOM_HEAP=1 SINGLE_THREADED_GC=1 HAVE_FFP_SHADER_SUPPORT=1
DEPTH_STENCIL_HACK=1 CIRCULAR_POOL_SPEEDHACK=1 ENABLE_LEGACY_PIPELINE=1
NO_SPLASHSCREEN=1
```

And it patches the library for N64 fog, plus bumps the FFP shader cache magic so
stale cached shaders are not reused:

```
sed -i "s|float fog_dist = coords.z / coords.w;|float fog_dist = coords.z;|" source/shaders/ffp_f.h
sed -i "s|#define FFP_SHADER_CACHE_MAGIC |#define FFP_SHADER_CACHE_MAGIC 99|" source/shared.h
```

Two general lessons:

- **record your vitaGL flag set in the README** — the library is configured by build
  flags, so "vitaGL 1.2" does not identify a binary
- **bump a cache magic when you patch a shader**, or the old compiled shader wins

## 5. Memory: the numbers a 3D emulator needs

| Consumer | DaedalusX64-class |
|----------|-------------------|
| recompiled-code cache | several MB, executable |
| ROM + RDRAM | guest RAM, often 8–64 MB |
| texture cache | tens of MB, in VRAM |
| shader/pipeline cache | MB |
| frame buffers | 2–3 × ~2 MB, CDRAM |

`HAVE_CUSTOM_HEAP=1` in vitaGL exists because emulators want their own allocator
policy over the library's.

## 6. Build file: keep the unstripped ELF

melonDS-Vita's Makefile is the cleanest example of the discipline:

```make
CFLAGS = $(INCLUDE) -D__PSP2__ -D__GNU__ -mfpu=neon -mcpu=cortex-a9 \
         -Wl,-q -O3 -g -ffast-math -fno-optimize-sibling-calls
CXXFLAGS = $(CFLAGS) -fno-exceptions -std=gnu++11

%.velf: %.elf
	cp $< $<.unstripped.elf      # ← symbols kept for crash dumps
	$(PREFIX)-strip -g $<
	vita-elf-create $< $@

$(TARGET).vpk: $(TARGET).velf
	vita-make-fself -s $< build/eboot.bin
	vita-mksfoex -s TITLE_ID=$(TITLE) -d ATTRIBUTE2=12 "$(TARGET)" param.sfo
	7z a -tzip ./$(TARGET).vpk -r ./build/sce_sys ./build/eboot.bin
```

`-fno-optimize-sibling-calls` matters for emulators: it keeps stack frames walkable,
which makes debugging and core dumps usable.

## 7. Emulator-specific UI is not optional

All three ship a proper menu layer: game selector with covers, per-game settings,
savestates, controller mapping, and a "download/update" screen in DaedalusX64's case.
An emulator without a frontend is not shippable, no matter how good the core is.

## 8. Checklist for porting an emulator

- [ ] does an ARM dynarec already exist (PSP-based ports usually have one)?
- [ ] is the guest GPU simple enough for a software rasteriser?
- [ ] what does the frontend need: selector, settings, savestates, input map?
- [ ] your vitaGL flag set recorded, and a cache magic bumped on shader patches?
- [ ] unstripped ELF archived per build?
- [ ] guest data path documented (`ux0:data/<emu>/`), with BIOS/ROM requirements
      stated plainly
- [ ] per-game compatibility list started on day one
