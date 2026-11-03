# 05 — Shader Toolchain

Three ways a shader gets onto the GPU. Pick deliberately — this decision shapes your
whole renderer.

---

## 1. The three paths

| Path | Tool | Runtime cost | Used by |
|------|------|--------------|---------|
| Offline Cg | `psp2cgc` → `.gxp` → `objcopy` into the binary | zero | Play!, PCSX AwO, gxmfun, anything with a fixed shader set |
| Runtime Cg | Cg source + `vitaShaRK`/`SceShaccCgExt` → program blob | ms per program, cached | Melee's GX permutations, UE-style material systems |
| GLSL | `vitaGL` translates GLSL at runtime | ms per program | GL-based ports (Android loaders) |

Runtime compilation has its own document now — API, options, logging and teardown in
[docs/40-runtime-shader-compiler.md](40-runtime-shader-compiler.md). Read that before
writing a compile wrapper by hand.

## 2. Offline: the simplest correct pipeline

```make
%.gxp: %.cg
	psp2cgc -profile sce_vp_psp2 -o $@ $<      # vertex
psp2cgc -profile sce_fp_psp2 -o $@ $<          # fragment

%.o: %.gxp
	arm-vita-eabi-objcopy -I binary -O elf32-littlearm --binary-architecture arm $< $@
```

Link the `.o` files; the shader bytes are then just symbols in your binary. PCSX
AwO goes one step further and generates a C array from the `.gxp` files via
`tools/embed_vita_postfx.py`, which is even easier to index at runtime.

**Use this for everything fixed.** A post-processing chain, a UI shader, a
skybox — none of them should ever compile at runtime.

## 3. Runtime: when you must

Prerequisites on device:

| Requirement | Why |
|-------------|-----|
| `ur0:data/libshacccg.suprx` | the actual shader compiler module |
| `libSceShaccCg_stub` link | SDK-side stub (present in `Sdk/lib`) |
| `SceShaccCgExt` | RE'd extended headers (`bythos14`) |
| `vitaShaRK` | friendlier wrapper over `sceShaccCg` |

Install order matters: `SceShaccCgExt` **then** `vitaShaRK`, from compatible
commits. Mismatched versions produce programs that compile "successfully" and
render garbage — a bug that costs days because it looks like a renderer bug.

Validate everything you load or generate:

```c
[[nodiscard]] auto ValidateGxmProgram(const void* program) noexcept -> bool {
    return sceGxmProgramCheck(static_cast<const SceGxmProgram*>(program)) >= 0;
}
```

## 4. Cg dialect notes for `sce_vp_psp2` / `sce_fp_psp2`

- entry point is `main`, parameters use semantics (`POSITION`, `COLOR`,
  `TEXCOORD0`, `TEXUNIT0`)
- `sampler2D` for textures; uniforms are declared inline in the parameter list
- output must be declared with a semantic and assigned
- no dynamic loops with variable bounds, no derivatives in vertex shaders
- precision matters: `float` is 32-bit on SGX, `half` is 16-bit and faster —
  Melee switched its TEV path between the two and bumped the cache version
  (`warm4.bin` → `warm5.bin`) because the compiled output changed

Example fragment shader (Play! `texture_f.cg`):

```c
void main(float4 fragment_color : COLOR,
          float2 texcoord      : TEXCOORD0,
          uniform sampler2D tex : TEXUNIT0,
          out float4 out_color  : COLOR)
{
    float4 texel_color = tex2D(tex, texcoord);
    float3 out_rgb = saturate(2 * texel_color.rgb * fragment_color.rgb);
    float  out_a   = saturate(2 * texel_color.a   * fragment_color.a);
    out_color = float4(out_rgb, out_a);
}
```

The `2 *` is not a typo — it reproduces the PS2 GS's colour-doubling blend. Shader
math reproduces platform quirks literally; "correct" is whatever the original did.

## 5. Permutation strategy

A UE-era engine can generate thousands of shader variants. Do not.

```
classify material  ──▶  one of N classes (tens, not thousands)
        │
        ▼
key = hash(class, alpha_mode, vertex_format, lighting_mode, fog, ...)
        │
        ▼
lookup RAM → writable cache → packaged cache → compile → store
```

If the class count is small, most keys hit the cache forever after the first run.

## 6. Debugging shader problems

| Symptom | Likely cause |
|---------|--------------|
| compiles, draws nothing | wrong attribute bindings, or output semantic typo |
| all black | `sceGxmProgramCheck` failed and you ignored it |
| flat colour | texture not bound, or wrong texture unit index |
| transposed texture | swizzle flag vs. data mismatch |
| banding on gradients | `half` precision where `float` was needed |
| crash at compile | out of shader-patcher cache memory |

## 7. Sources

- `Other Emulators/play-master/Source/gs/GSH_GXM/shader/*.cg` + its `CMakeLists.txt`
- `Other Emulators/PCSX_AwO-master/assets/postfx/**` + `tools/embed_vita_postfx.py`
- `Other Ports/melee-Vita-vita-port/platforms/vita/game/gxr_shader_cache.c`
- `Sdk/include/psp2/shacccg.h`
