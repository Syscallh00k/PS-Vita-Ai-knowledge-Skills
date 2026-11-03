---
name: vitagl-port
description: Get an OpenGL/GLES game rendering through vitaGL — API triage, init order, pool sizing, shader binding, and the failure modes that look like logic bugs.
when: The game's renderer calls gl*, or a vitaGL port shows a black screen / missing textures.
---

# Skill: vitaGL Port

## Step 1 — Find out what GL the game actually uses

```bash
grep -rn "gl30\|gdx.gl30\|glTexStorage\|glDrawBuffers\|glMapBufferRange" src/ | head -50
```

| Found | Verdict |
|-------|---------|
| GL 1.x fixed function | good — vitaGL's FFP is complete enough |
| GLES2 / GL 2.x | good — this is the design target |
| GL 3.x features (texture arrays, integer textures, MRT, `gl30`) | rewrite that path or reconsider the port |

## Step 2 — Check for stale API names

Grep the source for the retired names in doc 29 §1 (`vglSetVertexPoolSize`,
`vglUseVram`, `vglStartRendering`, `HAVE_GLSL_SUPPORT`, …). If a symbol does not
exist in `source/vitaGL.h`, it is gone — do not add it back.

## Step 3 — Size memory before init

Order matters, and getting it wrong is a boot crash with no log:

1. pick `_newlib_heap_size_user`
2. set the vitaGL `ram_threshold` so heap + stacks + pools fit the budget
3. call `vglInit*` last

`vglInit` forces 4× MSAA — pass `SCE_GXM_MULTISAMPLE_NONE` if you want the VRAM back.

**Gate:** the app boots and survives a `malloc`-heavy frame. If every allocation
fails, the heap was larger than free MAIN.

## Step 4 — Init order

```c
vglSetupRenderTargetScenesNum(2, 2);
vglSetShaderCachePath("ux0:data/<app>/shader_cache");
vglSetupRuntimeShaderCompiler(SHARK_OPT_DEFAULT, GL_TRUE, GL_TRUE, GL_TRUE);
vglInitExtended(0, 960, 544, ram_threshold, SCE_GXM_MULTISAMPLE_NONE);
vglWaitVblankStart(GL_TRUE);
```

Then GL setup, then per frame: draw → `vglSwapBuffers(GL_FALSE)`.

Every GL call after init must come from the **same thread**.

## Step 5 — Shaders

- bind attribute locations **before** `glLinkProgram`
- remember `glCompileShader` is a no-op: errors surface at link
- keep fragment uniforms ≤ 16 vectors (a real, low limit)
- shaders compile through `libshacccg.suprx`; without it you get
  `GL_INVALID_OPERATION` and a black screen
- enable the shader cache — otherwise every launch recompiles and stutters

## Step 6 — Geometry limits

`glDrawArrays` caps at **49152 vertices** through an internal index buffer. Larger
draws give garbage geometry with no log. Either chunk the draw or use
`glDrawElements` with your own indices.

## Step 7 — Diagnosis

| Symptom | Check |
|---------|-------|
| black screen | `libshacccg.suprx` present? compile at link? swap called? |
| every texture vanishes mid-run | a sticky `vgl_error` from `glPixelStorei` |
| geometry corrupt past N vertices | the 49152 cap |
| GL calls do nothing | called from the wrong thread |
| depth reads all zero | `glReadPixels(GL_DEPTH_COMPONENT)` unsupported |
| one-texel offset in a LUT | `gl_FragCoord` is the pixel centre |
| slow with no obvious cost | a texture format falling back to CPU conversion |

## Step 8 — Verify, then stop tuning

Once it renders, check the frame in the **worst** scene at 480×272. If it holds,
move on — micro-optimising the GL path before the game is playable is the classic
way to spend a month for nothing.

## Never

- never bundle `libshacccg.suprx`
- never call GL from a second thread "just for loading"
- never trust `glCompileShader`'s return value for correctness
