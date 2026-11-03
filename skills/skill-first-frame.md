---
name: first-frame
description: Bring a GXM renderer up in five verified stages so a black screen becomes a diagnosis instead of a mystery.
when: Nothing renders, something renders wrong, or a new renderer needs standing up.
---

# Skill: First Frame

## The stages

| Stage | Proves | Failure looks like |
|-------|--------|--------------------|
| 1. `vita2d` clear | display path | black screen, no crash |
| 2. hardcoded `.gxp` triangle | shader patcher | black screen |
| 3. swizzled textured quad | texture upload | flat colour / transposed |
| 4. blend + depth + 1000 draws | state handling | wrong blend, z-fighting |
| 5. batched mesh | throughput | 15 fps |

Do not skip 1–3. Every project that skipped them spent a week on "why is it black".

## Stage 1 — display

```c
vita2d_init();
vita2d_set_clear_color(0xFF202030);
vita2d_start_drawing();
vita2d_clear_screen();
vita2d_end_drawing();
vita2d_swap_buffers();
```

**Gate:** colour on screen. If not, the problem is init, memblocks or the display
buffer — nothing to do with your renderer.

## Stage 2 — shader patcher

1. `sceGxmCreateContext`
2. `sceGxmCreateRenderTarget` at 960×544
3. `sceGxmShaderPatcherCreate` with a cache buffer and your program list
4. `sceGxmShaderPatcherRegisterProgram` → id
5. `sceGxmShaderPatcherCreateVertexProgram` / `CreateFragmentProgram`
6. `sceGxmBeginScene` → set programs → `sceGxmDraw` → `sceGxmEndScene` →
   `sceGxmDisplayQueueAddEntry`

Validate the program before trusting it:

```c
[[nodiscard]] auto ValidateGxmProgram(const void* program) noexcept -> bool {
    return sceGxmProgramCheck(static_cast<const SceGxmProgram*>(program)) >= 0;
}
```

**Gate:** a triangle. If it fails here, the `.gxp` is wrong, not the drawing code.

## Stage 3 — texture

- destination memory: CDRAM or an uncached memblock
- choose `SCE_GXM_TEXTURE_SWIZZLE` + Morton data **or** `LINEAR` + linear data
- test with a 2-colour checkerboard — a transpose is obvious immediately

## Stage 4 — state

- blend lives in the fragment program → cache `(program, blend)` pairs
- depth func/write/bias are dynamic → free to change
- alpha test does not exist → build it in against a threshold uniform
- resolve uniforms once per program, never per draw

## Stage 5 — throughput

- interleaved vertices, 4-byte aligned, ≤ 32 B/vertex, `_Static_assert` the layout
- `U16` indices under 65536 verts, triangles as lists
- sort draws by (program, texture)

## Diagnosis table

| Symptom | First thing to check |
|---------|----------------------|
| black, no crash | did you ever call `EndScene` + display queue? |
| black with Stage 2 only | `sceGxmProgramCheck` result |
| flat colour | texture bound? correct unit index? |
| transposed | swizzle flag vs. data layout |
| flicker | not double-buffering, or presenting before `EndScene` |
| works in Vita3K, not on hardware | memory type (cached vs uncached) |

## Never

- never add engine data before stage 3 passes
- never ignore a negative return from any `sceGxm*` call — log it
