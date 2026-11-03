# 04 — GXM Rendering

`SceGxm` is a shader-patcher API, not OpenGL. No fixed pipeline, no state sorting,
no automatic anything.

---

## 1. Objects

```
SceGxmContext
SceGxmShaderPatcher
 ├── registerProgram(id, gxp)          -> SceGxmShaderPatcherId
 ├── createVertexProgram(id, attrs)    -> SceGxmVertexProgram
 └── createFragmentProgram(id, blend)  -> SceGxmFragmentProgram

SceGxmRenderTarget         (offscreen; MSAA samples fixed at creation)
SceGxmDepthStencilSurface  (depth + stencil)
SceGxmTexture              (format, swizzle, mips)
```

## 2. A frame

```c
sceGxmBeginScene(g_context, 0, g_target, NULL, &g_depth, &g_sync0, &g_sync1);
sceGxmSetVertexProgram(g_context, g_vp);
sceGxmSetFragmentProgram(g_context, g_fp);
sceGxmReserveVertexDefaultUniformBuffer(g_context, &ubo);
/* write uniforms, set streams, set textures */
sceGxmDraw(g_context, SCE_GXM_PRIMITIVE_TRIANGLES, SCE_GXM_INDEX_FORMAT_U16,
           g_indices, count);
sceGxmEndScene(g_context, NULL, &g_display_sync);
sceGxmDisplayQueueAddEntry(g_display_sync, 0, 0);
```

## 3. Cache parameters per program, not per draw

```c
typedef struct GxrProgram {
    uint64_t hash;
    SceGxmShaderPatcherId id;
    const SceGxmProgram* program;
    const SceGxmProgramParameter* uniforms[8];
    bool failed;
} GxrProgram;
```

`sceGxmProgramFindParameterByName` walks a table. Calling it inside a draw loop is
the single most common self-inflicted performance bug on this platform.

## 4. State that is dynamic vs. baked

| State | Dynamic? |
|-------|----------|
| depth func, depth write, depth bias | yes (`sceGxmSetFrontDepthFunc` etc.) |
| viewport, scissors | yes |
| blend | **no** — it is part of the fragment program |
| alpha test | **no** — implement in the shader against a threshold uniform |
| cull mode | yes |

Consequence: cache fragments per `(program, blend)` pair, and treat "material" as a
shader permutation, not a state block.

```c
typedef struct GxrFragment {
    SceGxmBlendInfo blend;
    SceGxmFragmentProgram* fragment;
    struct GxrFragment* next;
} GxrFragment;
```

## 5. Render targets and resolution

- `sceGxmCreateRenderTarget` takes a `SceGxmRenderTargetParams` with `width`,
  `height`, `scenesPerFrame`, `multisampleMode`. **Samples cannot change after
  creation** — changing MSAA means relaunching (`sceAppMgrLoadExec`), which is what
  PCSX AwO does.
- Display is fixed at 960×544. Render smaller and upscale.
- Alignment rules that actually bite: width % 16 == 0, height % 8 == 0.

## 6. Resolution scaling policy

Frames that need exact texel alignment must be rendered at native size, not scaled:

| Reason | Example |
|--------|---------|
| render-target readback | post-processing passes that sample the previous frame |
| copy-to-texture | shadow maps, mirrors |
| video/YUV upload | `sceAvPlayer` frames |
| UI | text at non-integer scale looks broken |

Melee's port carries this as flags (`COPY`, `READBACK`, `SHADOW`, `THP`,
`DEBUG_UI`) and forces a native redraw rather than faking it.

## 7. Textures

- format enum in `Sdk/include/psp2/gxm.h` — the authoritative list
- swizzle is a **flag plus a data layout**: `SCE_GXM_TEXTURE_SWIZZLE` with Morton
  data, or `LINEAR` with linear data; never mix
- uncached memory (`USER_RW_UNCACHE`) or CDRAM for anything the GPU touches
- `sceGxmTextureSetData` for a raw pointer, `sceGxmTextureInitLinear` for pitch

## 8. Bring-up order (do not skip)

1. `vita2d` clear to a colour → proves display path
2. one hardcoded `.gxp` triangle → proves the patcher
3. one swizzled textured quad → proves upload
4. blend + depth + 1000 draws → proves state handling
5. batched mesh → proves throughput
6. engine data

## 9. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/game/gx_render.c` (130 KB, the
  most complete GXM renderer in the workspace)
- `Other Emulators/play-master/Source/gs/GSH_GXM/gxm.cpp`
- `Sdk/include/psp2/gxm.h`
