# 29 — vitaGL

OpenGL ES 2.0 (plus a good chunk of fixed-function GL 1.x) translated onto GXM.
If a port's renderer calls `gl*`, this is the layer it lands on.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/vitagl.md`, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Stale names — do not write these

Tutorials, old READMEs and model memory still emit them. They no longer exist:

| Old | Now |
|-----|-----|
| `HAVE_SHARK=1/2` | removed 2021 — runtime compiler always on |
| `vglEnableRuntimeShaderCompiler` | removed 2022 |
| `HAVE_GLSL_SUPPORT=1/2` | removed 2026-04 — translator always on |
| `vglSetVertexPoolSize` | `vglSetCircularPoolSize(uint32_t)` (default 32 MiB) |
| `vglUseVram` | removed 2026-05 (memory rework); `vglUseVramForUSSE` remains |
| `vglSetupDisplayRenderTarget` | `vglSetupRenderTargetScenesNum(display, fbo)` |
| `VGL_MEM_SLOW` | `VGL_MEM_PHYCONT` |
| `vglStartRendering` / `vglStopRendering` | `vglSwapBuffers(GL_FALSE)` |
| `vglDrawObjects(mode, count, GL_TRUE)` | `vglDrawObjects(mode, count)` |
| Cg via `glCreateShader(GL_VERTEX_SHADER)` | `GL_CG_VERTEX_SHADER_EXT` / `GL_CG_FRAGMENT_SHADER_EXT` |
| raw `.gxp` via `glShaderBinary` | `vglShaderGxpBinary` |

Rule: if an API in old port code does not exist, grep `source/vitaGL.h` in the
vitaGL checkout instead of guessing.

## 2. What it covers

- **Fixed function**: matrix stacks, 8 lights, fog, texenv including COMBINE/dot3,
  7 clip planes, `glBegin/glEnd` immediate mode, `GL_QUADS`, `glInterleavedArrays`,
  `glPushAttrib`. Display lists need `HAVE_DISPLAY_LISTS=1`. Only 2 FFP texture
  units unless `HAVE_HIGH_FFP_TEXUNITS=1`.
- **Shaders**: GLSL ES 1.00 style, Cg, or precompiled GXP. No geometry,
  tessellation, compute, or transform feedback.
- **Not present**: `glTexImage3D`, `glTexStorage*`, `glFenceSync`, `glGetTexImage`,
  `glLogicOp`, `glBindFragDataLocation`, `glVertexAttribIPointer`, `glDrawBuffers`
  (single colour attachment, no MRT). `glBlendColor` / `glBlendFunci` are also
  missing from the archive — pass a blend colour as a uniform and do the arithmetic
  in the fragment shader.
- **Limits that bite**: max texture/renderbuffer/viewport 4096; **fragment uniform
  vectors 16**; vertex uniform vectors 128; varyings 8; texture units 16.
- **The 49152-vertex trap**: `glDrawArrays` / `glDrawArraysInstanced` draw through
  one built-in index buffer of 0xC000 vertices. Larger counts read past it and give
  garbage geometry with **no log in a `NO_DEBUG` build**. `glDrawElements` has no
  such cap.
- **Depth is not stored** unless the library is built with `STORE_DEPTH_STENCIL=1`;
  `glReadPixels(GL_DEPTH_COMPONENT)` raises `GL_INVALID_ENUM` and leaves your buffer
  untouched — a value-initialised array becomes an all-zero depth image silently.
  Use `vglTexImageDepthBuffer` and sample it.
- **`gl_FragCoord.xy` is the pixel centre** (x + 0.5). A LUT lookup written as
  `(gl_FragCoord.x + 0.5) * invW` is one texel off with NEAREST; use
  `gl_FragCoord.x * invW`.
- **Thread rule**: every GL call must come from the thread that called `vglInit*`.
  `DEBUG_THREAD_SAFENESS=1` reports violations.

Native texture formats: RGBA8, RGB888, RGB565, RGBA4444, RGB5A1, L8, A8, LA8, RG8,
R8, RGBA16F, P4/P8 palettes, DXT1/3/5, RGTC, PVRTC v1/v2, ETC1, YUV420 planar.
Anything else converts on the CPU — slow, so convert offline instead. NPOT is
advertised but NPOT mipmaps are broken upstream.

---

## 3. Initialisation order

```c
vglSetupRenderTargetScenesNum(2, 2);
vglSetShaderCachePath("ux0:data/<app>/shader_cache");   /* needs HAVE_SHADER_CACHE=1 */
vglUseLowPrecision(GL_TRUE);                             /* float -> half */
vglSetupRuntimeShaderCompiler(SHARK_OPT_UNSAFE, GL_TRUE, GL_TRUE, GL_TRUE);
vglInitExtended(0, 960, 544, 24 * 1024 * 1024, SCE_GXM_MULTISAMPLE_4X);
vglWaitVblankStart(GL_TRUE);
/* GL setup, then per frame: draw, vglSwapBuffers(GL_FALSE) */
```

Calls that must happen **before** `vglInit*`: `vglSetParamBufferSize`,
`vglSetVertexBufferSize`, `vglSetFragmentBufferSize`, `vglSetUSSEBufferSize`,
`vglSetupShaderPatcher`, `vglSetupGarbageCollector`, `vglSetShaderCachePath`,
`vglSetCircularPoolSize`, `vglUseCachedMem`.

`vglUseLowPrecision(GL_TRUE)` is a real trade: half floats carry 11 significant
bits, so UVs on textures ≥1024 texels and large world positions lose accuracy. Test
per game.

Zero-copy buffers, worth knowing for emulator frontends: allocate with `vglMalloc`
(GPU-mapped), fill on the CPU, attach with `vglBufferData(target, ptr)` instead of
`glBufferData`, free with `vglLazyFree` (waits until the GPU is done). Flycast-vita
builds its per-frame vertex and index lists this way.

## 4. Memory: the threshold is the whole design

`vglInitExtended(legacy_pool, w, h, ram_threshold, msaa)`. The RAM pool becomes
"free user memory at init **minus** `ram_threshold`" — so the threshold is what you
leave for the newlib heap, stacks and libraries.

Order of operations:

1. decide `_newlib_heap_size_user`
2. set `ram_threshold` so heap + stacks + pools fit the memory budget
3. only then call `vglInit*`

Real values: VitaKotor2 used 160 MiB newlib + an 80 MiB vitaGL threshold. bc2_vita
used 192 MiB newlib + `vglInitWithCustomThreshold(0, 960, 544, 8 MiB, 0, 0, 0, 4x)`.

Defaults worth overriding: `vglInit` forces 4× MSAA (pass `SCE_GXM_MULTISAMPLE_NONE`
to save VRAM), circular pool 32 MiB, triple buffering on.

## 5. Shaders and `libshacccg.suprx`

vitaGL compiles through vitaShaRK → Sony's `libshacccg.suprx`, which must exist at
`ur0:data/libshacccg.suprx`. It is Sony's file: **never bundle it**, and say so in
the user README.

Binding mode defaults to `VGL_MODE_POSTPONED`: `glCompileShader` is a no-op,
`GL_COMPILE_STATUS` is always true, and **GLSL really compiles at
`glLinkProgram`** — which is where your errors appear. Bind attribute locations
before link or you will spend an evening on missing vertex data.

## 6. Common problems

| Symptom | Cause |
|---------|-------|
| every texture disappears after an innocuous call | sticky `vgl_error` from `glPixelStorei` with an unsupported pname |
| black screen, no crash | no `libshacccg.suprx`; `glCompileShader` returned `GL_INVALID_OPERATION` |
| garbage geometry past N thousand vertices | the 49152 `glDrawArrays` cap |
| GL calls silently do nothing after a thread change | called outside the init thread |
| one-column offset in a LUT/post shader | `gl_FragCoord` centre convention |
| all-zero depth image | `glReadPixels(GL_DEPTH_COMPONENT)` is not supported |
| right in one scene, broken in another | a texture format fell back to a CPU conversion path |

