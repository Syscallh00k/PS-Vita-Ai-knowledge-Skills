---
name: shader-runtime-compile
description: Compile shaders on the console with vitaShaRK — init variants, options, logging, patcher registration, and a cache that stops the stutter.
when: Shaders are generated at runtime, or you need to cache compiled GXM programs on device.
---

# Skill: Runtime Shader Compile

## Step 1 — Decide if you need it at all

| Shader set | Do this instead |
|------------|-----------------|
| fixed (post-FX, UI, skybox, one material) | offline `psp2cgc` → `.gxp` → `objcopy` (doc 05) |
| keyed by GPU state, unbounded | runtime compile + cache (this skill) |
| GL game | vitaGL does it for you |

Offline is always better. Only generate what you cannot enumerate.

## Step 2 — Init

```c
if (shark_init(NULL) < 0) {                 /* ur0:data/libshacccg.suprx */
    LogWrite("shader compiler missing");
    FatalError("Install libshacccg.suprx in ur0:data/");
}
```

- `shark_init` → with `SceShaccCgExt` extensions (needed for full Cg)
- `shark_init_simple` → no extensions, **what makes Vita3K work**
- pick at build time if you ship both paths

**Gate:** `shark_init` returns ≥ 0. If not, the module is missing or SceShaccCgExt is
out of sync.

## Step 3 — Log every compile

```c
shark_install_log_cb(OnShaderLog);
shark_set_warnings_level(SHARK_WARN_MEDIUM);
```

Without the callback, a failed compile is indistinguishable from a rendering bug.
Route the callback into your normal log with the reported line number.

## Step 4 — Compile with the right options

```c
uint32_t size = (uint32_t) strlen(source);
SceGxmProgram* program = shark_compile_shader_extended(
    source, &size, SHARK_FRAGMENT_SHADER,
    SHARK_OPT_FAST,   /* SLOW, SAFE, DEFAULT, FAST, UNSAFE */
    /*fastmath*/ 1, /*fastprecision*/ 0, /*fastint*/ 0);
```

- `fastprecision` demotes `float` to `half` — a big win, and a real risk for UVs on
  textures ≥1024 and for world positions. Test per shader.
- always `shark_clear_output()` after you have copied what you need

## Step 5 — Register and draw

```c
SceGxmShaderPatcherId id;
sceGxmShaderPatcherRegisterProgram(g_patcher, program, &id);
/* then CreateVertexProgram / CreateFragmentProgram, per blend state */
```

Load a `.gxp` from disk the same way — the patcher does not care where the program
came from.

## Step 6 — Cache, or stutter forever

```
key = hash(state that changes compiled code)
  RAM table → ux0:data/<game>/shaders/<key>.gxp → app0:/shaders/<key>.gxp → compile
```

Write the program bytes straight to disk (the sample in doc 40 does exactly this), and
on a hit load the file and validate with `sceGxmProgramCheck`. Bump a cache version
constant whenever you change shader source or compiler flags — otherwise stale
programs win and you debug a ghost.

## Step 7 — Teardown

```c
shark_end();   /* releases the compiler, disables extensions, unloads the module */
```

Skipping this leaks the module and its allocations across a relaunch.

## Step 8 — Diagnose

| Symptom | Cause |
|---------|-------|
| every compile fails | `libshacccg.suprx` missing |
| compiles, draws nothing | wrong attribute bindings / semantics |
| some shaders fail only in release | a fast flag changed codegen |
| first-visit stutter | no cache, or cache version bumped every build |
| crash inside the compiler | out of memory — set custom allocators, free the output |

## Never

- never compile on the render thread mid-frame
- never bundle `libshacccg.suprx`
- never assume `GL_COMPILE_STATUS`-style success from vitaGL means the GXM compile
  worked
