# 21 — Porting Emulator Frontends

Emulators are the most portable kind of "port": the guest machine is code you
already control, so the work is all host-side.

Two references: `PCSX_AwO` (PS1, Makefile, vitaGL) and `play-master` (PS2, CMake,
GXM backend).

---

## 1. The two-part structure

```
 core (libretro-style)  ──▶  host frontend  ──▶  display / audio / input
```

Both references keep a **core** that is independent of the host, and a **frontend**
that owns video, audio, input, menus, and settings. That split is why they can also
run in RetroArch.

| Project | Core | Frontend |
|---------|------|----------|
| PCSX AwO | `pcsx_rearmed_libretro_vita.a` | `frontend/vita/standalone.c` |
| Play! | `PlayCore` | `Source/ui_vita/main.cpp` |

## 2. PCSX AwO: what a Makefile build looks like

```make
CFLAGS := -O3 -DVITA -mcpu=cortex-a9 -mtune=cortex-a9 -mfpu=neon -marm \
          -fsingle-precision-constant -mword-relocations -ffast-math \
          -fno-unwind-tables -fno-asynchronous-unwind-tables
LDFLAGS := -Wl,-q
LIBS    := -lSDL3 -limgui $(VITAGL_DIR)/libvitaGL.a -lvitashark -lSceShaccCgExt \
           -ltaihen_stub -lkubridge_stub -lmathneon ...
```

Notable choices:

- `-marm` (not Thumb-2): bigger code, faster hot loops on A9
- `-fsingle-precision-constant` + `-ffast-math`: huge NEON win, safe in host code
- `-fno-unwind-tables`: smaller binary, cheaper prologues
- vitaGL linked statically from its build dir rather than installed

## 3. Play!: the backend-as-a-class pattern

`Source/gs/` holds five graphics backends: Direct3D9, OpenGL, OpenGLWin32, Vulkan,
GXM. Adding a host platform means adding one class, not editing the core.

`GSH_GXM` is small and complete: command handling, texture upload, fragment program
cache, and six `.cg` shaders compiled at build time.

```cmake
add_custom_command(OUTPUT "${shader_we}.gxp"
    COMMAND $ENV{PSP2CGC} -profile sce_vp_psp2 "${shader}" -o "${shader_we}.gxp")
add_custom_command(OUTPUT "${shader_we}_gxp.o"
    COMMAND arm-vita-eabi-objcopy -I binary -O elf32-littlearm \
            --binary-architecture arm "${shader_we}.gxp" "${shader_we}_gxp.o")
```

## 4. JIT: the one thing that needs special memory

Emulators need executable memory for recompiled guest code. Two approaches:

| Approach | Requirement | Used by |
|----------|-------------|---------|
| VM block + `sceKernelSyncVMDomain` | none beyond the SDK | Play! |
| Patch existing code pages via kubridge | `kubridge.skprx`, `UNSAFE` SELF | loaders/hookers |

Play! allocates a 16 MB VM block and bumps an offset per allocation — no kernel
plugin needed. Prefer this whenever you control the code.

## 5. Frontend responsibilities

1. **Boot**: parse launch args (`sceAppMgrGetAppParam`), or show a file chooser
2. **Video**: present the emulated framebuffer (software blit or GPU path)
3. **Audio**: open a port at the emulator's rate, stream mixed frames
4. **Input**: map `SceCtrl` to the emulated controller(s), including analog ranges
5. **Menus**: pause menu, settings, per-game overrides
6. **Saves**: states and memory cards on `ux0:`
7. **Exit**: clean teardown (`done:` label, free blocks, close ports)

## 6. Presentation paths compared

| Path | How | When |
|------|-----|------|
| Software framebuffer | emulator writes RGB565 → LUT → `sceDisplaySetFrameBuf` | CPU rasterizers, exact pixel output |
| Texture upload | framebuffer → texture → quad | needs scaling/filters like CRT shaders |
| Native GPU render | emulator draws with vitaGL/GXM directly | "High Octane"-style renderers |

PCSX AwO supports all three, and uses a precomputed `xmap[]`/`ymap[]` scale table so
the blit costs no per-pixel math.

## 7. Per-game settings are not optional

Emulated titles differ wildly. Key settings by disc serial (`SLUS_…`, `SCES_…`) so
one title can pick a different renderer, frame-generation mode, or memory-card
policy. PCSX AwO also uses this to persist per-game overrides between sessions.

## 8. Honest performance ceilings

| Guest | Verdict on Vita |
|-------|-----------------|
| PS1 | playable at 60 fps with a good renderer |
| N64 | borderline; needs dynarec and frame skipping |
| PS2 | only barely functional; not playable |
| PSP | not viable without heavy dynarec work |
| PS3 / Xbox 360 | not viable |

One data point worth noting: **melonDS-Vita runs a DS emulator on this hardware using
an ARM interpreter and software rasterisers**, with no dynarec and no shader
compiler. When the guest GPU is simple, a software rasteriser plus a `vita2d` blit
can be both easier and more accurate than a shader pipeline.

Full anatomy of an emulator port — platform directory, dynarec vs interpreter,
memory budget, frontend requirements — is in
[docs/44-emulator-port-anatomy.md](44-emulator-port-anatomy.md).

## 9. Sources

- `Other Emulators/PCSX_AwO-master/Makefile.vita`, `frontend/vita/standalone.c`,
  `tools/embed_vita_postfx.py`
- `Other Emulators/play-master/Source/ui_vita/*`, `Source/gs/GSH_GXM/*`
