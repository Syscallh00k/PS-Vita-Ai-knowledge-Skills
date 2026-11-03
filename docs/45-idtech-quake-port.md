# 45 — id Tech Ports (Quake II, Wolfenstein 3D)

The most-repeated port shape in the Vita homebrew scene: an open-source id Tech game
with a thin platform directory named `psp2/`.

> Sources: `Other Ports/vitaQuakeII-master`, `Other Ports/vitaWolfen-master`.
> See [CREDITS](../CREDITS.md).

---

## 1. The platform directory

vitaQuakeII is the textbook layout:

```
source/            upstream Quake II
psp2/
  vid_psp2.c       window, video modes, GL context, post-FX
  glimp_psp2.c     GL entry points, vglInit, swap, resolution
  in_psp2.c        input
  snddma_psp2.c    audio DMA
  sys_psp2.c       timing, sleep, console, quit
  net_psp2.c       sockets
  glob.c           path/config helpers
```

Six files, one per subsystem the engine expects from the OS. **This is the entire
port.** Everything else is the game, untouched.

If a codebase has clean platform seams (id Tech does — `vid_*.c`, `snddma_*.c`,
`sys_*.c` per OS), a port is days, not months.

## 2. Video: the smallest correct setup

```c
int msaa = 0, postfx = 0, scr_width = 960, scr_height = 544;

if (msaa == 2)      vglInitExtended(0, scr_width, scr_height, 0x1000000, SCE_GXM_MULTISAMPLE_2X);
else if (msaa == 4) vglInitExtended(0, scr_width, scr_height, 0x1000000, SCE_GXM_MULTISAMPLE_4X);
else                vglInitExtended(0, scr_width, scr_height, 0x1000000, SCE_GXM_MULTISAMPLE_NONE);
...
vglSwapBuffers(isKeyboard);
```

Points worth copying:

- `0x1000000` (16 MiB) `ram_threshold` — sized so the rest of the heap survives
- MSAA chosen **before** init, because a render target's sample count is fixed
- post-FX done as a full-screen quad in 960×544 space (`fx_vertices[...] = 960/544`)
- `vglSwapBuffers(isKeyboard)` — the flag tells vitaGL to draw the system keyboard
  overlay when it is open, so text entry composites correctly

## 3. Audio DMA shim

`snddma_psp2.c` is only 3 KB: the engine hands you a ring buffer, you submit it to
`SceAudioOut` and report the write position. Every id Tech-era port has this file and
they are all about the same size.

## 4. Engine settings become CVars, not menus

Quake-lineage engines already have a cvar/config system, so the port exposes
settings the engine understands:

```
vid_fullscreen  vid_gamma  scr_viewsize  vid_ref (REF_OPENGL)  ...
```

`CL_WriteConfiguration()` persists them. You get a settings system for free — do not
build one on top.

## 5. Link line

```make
LIBS = -lvitaGL -lvorbisfile -lvorbis -logg -lspeexdsp -lmpg123 -lmathneon \
       -lSceLibKernel_stub -lSceAppMgr_stub -lSceSysmodule_stub -ljpeg \
       -lSceCtrl_stub -lSceTouch_stub -lm -lSceNet_stub -lSceNetCtl_stub \
       -lSceAppUtil_stub -lc -lScePower_stub -lSceMotion_stub \
       -lSceCommonDialog_stub -lSceAudio_stub -lSceGxm_stub -lSceDisplay_stub \
       -lvitashark -lSceShaccCgExt -ltaihen_stub -lSceShaccCg_stub \
       -lSceKernelDmacMgr_stub
```

Note the codec set: `vorbis` + `speexdsp` + `mpg123` covers Ogg music, reverb/echo
DSP and MP3 audio without pulling FFmpeg.

## 6. Wolfenstein 3D: the minimal case

vitaWolfen is even smaller — one runtime (`id Tech 0`) and a **launcher** that builds
multiple variants:

```make
all:
	make WMODE=0 MISSIONPACK=0 -C source
	make WMODE=1 MISSIONPACK=0 -C source
	make WMODE=2 MISSIONPACK=0 -C source
	make WMODE=3 MISSIONPACK=0 -C source
	make WMODE=3 MISSIONPACK=2 -C source
	make WMODE=3 MISSIONPACK=3 -C source
	make -C launcher
	cp launcher/vitaWolfen.vpk vitaWolfen.vpk
```

`WMODE` selects the game (Wolf3D / Spear of Destiny), `MISSIONPACK` selects extra
content. One binary per configuration, and a launcher bubble that picks between them.

That is the pattern for "several related data sets, one engine": build per-title
variants rather than adding a data selector inside the engine.

## 7. What id Tech ports teach that transfer everywhere

| Lesson | Applies to |
|--------|-----------|
| A per-subsystem platform directory is the whole port | any engine with clean seams |
| Use the game's own config system instead of building one | any game with cvars/ini |
| Codec set: vorbis + mpg123 + speexdsp beats FFmpeg | every audio port |
| MSAA/resolution must be picked before GL init | every vitaGL port |
| Build per-title variants instead of a runtime selector | multi-dataset ports |
| `isKeyboard` flag on swap for text entry | any port with on-screen keyboard |

## 8. Effort estimate

For an engine of this era with existing Linux/SDL platform files: **a fortnight to
first frame, a month to playable**, mostly spent on input feel, save paths and audio
timing. For a 2020s engine with no platform seam: multiply by a lot.
