# 40 — Runtime Shader Compiler (vitaShaRK)

The library that turns Cg source into a GXM program **on the console**, at runtime.
Everything that generates shaders dynamically goes through it.

> Sources: `Other Emulators/vitaShaRK-master` (Rinnegatamante, LGPL-3.0),
> `Other Emulators/gxmfun-master`, `Other Emulators/PCSX_AwO-master`.
> See [CREDITS](../CREDITS.md).

---

## 1. What it actually does

```
your Cg source
     │
     ▼
shark_compile_shader_extended()      ── fills SceShaccCgCompileOptions
     │
     ▼
sceShaccCgCompileProgram()           ── Sony's compiler, loaded from
     │                                  ur0:data/libshacccg.suprx
     ▼
SceShaccCgCompileOutput
     │  output->program / output->programSize
     ▼
SceGxmProgram  ──▶ sceGxmShaderPatcherRegisterProgram() ──▶ draws
```

`libshacccg.suprx` is Sony's module from the PSM runtime. vitaShaRK loads it with
`sceKernelLoadStartModule` — you never link it, you load it.

## 2. Init, and the two variants

```c
int shark_init(const char *path);          /* enables SceShaccCgExt extensions */
int shark_init_simple(const char *path);   /* no extensions — Vita3K compatibility */
```

- `path == NULL` → defaults to `ur0:data/libshacccg.suprx`
- `shark_init` calls `sceShaccCgExtEnableExtensions()` (requires **SceShaccCgExt**)
- `shark_init_simple` skips it, which is what makes builds run under Vita3K
- both call `sceShaccCgSetDefaultAllocator(malloc, free)` — replaceable through
  `shark_set_allocators()` to keep the compiler out of your game heap

Teardown: `shark_end()` → `sceShaccCgReleaseCompiler()`,
`sceShaccCgExtDisableExtensions()`, `sceKernelStopUnloadModule()`. Skipping the
unload leaks the module and its allocations across relaunches.

## 3. Compiling

```c
typedef enum shark_opt {          /* GCC-style optimisation levels */
    SHARK_OPT_SLOW,     /* O0 */
    SHARK_OPT_SAFE,     /* O1 */
    SHARK_OPT_DEFAULT,  /* O2 */
    SHARK_OPT_FAST,     /* O3 */
    SHARK_OPT_UNSAFE    /* Ofast */
} shark_opt;

SceGxmProgram* shark_compile_shader_extended(
    const char* src, uint32_t* size, shark_type type,
    shark_opt opt, int32_t use_fastmath, int32_t use_fastprecision, int32_t use_fastint);

SceGxmProgram* shark_compile_shader(const char* src, uint32_t* size, shark_type type);
```

- `size` is **in/out**: pass the source length, receive the program length
- the returned buffer belongs to the compiler until `shark_clear_output()`
- `use_fastprecision` / `use_fastint` demote to half/short — a real speed win and a
  real accuracy risk on UVs and world positions
- `SHARK_OPT_UNSAFE` with all three fast flags is what PCSX AwO uses for a post-FX
  chain, where precision does not matter

Minimal working example (`samples/sample1`):

```c
const char g_fragment_shader[] =
    "float4 main(uniform float4 u_clear_color) : COLOR\n"
    "{\n"
    "    return u_clear_color;\n"
    "}";

if (shark_init(NULL) < 0) return -1;               /* needs libshacccg.suprx */

uint32_t size = sizeof(g_fragment_shader) - 1;
SceGxmProgram* program = shark_compile_shader(g_fragment_shader, &size,
                                              SHARK_FRAGMENT_SHADER);
if (program != NULL) SaveGxp(program, size, "ux0:data/clear_f.gxp");
shark_clear_output();
```

## 4. Logging — turn it on once, then leave it on

```c
void shark_install_log_cb(void (*cb)(const char* msg, shark_log_level level, int line));
void shark_set_warnings_level(shark_warn_level level);   /* SILENT..MAX */
void shark_set_locale(shark_locale locale);
```

The compiler's own messages are the only useful output from a failed compile. Route
them into your log file with the line number. A silent compile failure looks exactly
like a black screen.

Also available: `shark_get_internal_compile_output()` for the raw
`SceShaccCgCompileOutput`, and `shark_set_shader_association_path()` to dump
association files that Razor (the official toolchain) can read.

---

## 5. Turning compiled programs into drawable ones

A `SceGxmProgram*` is not drawable. The full chain:

```c
SceGxmShaderPatcherId id;
sceGxmShaderPatcherRegisterProgram(g_patcher, program, &id);

SceGxmVertexProgram* vp;
sceGxmShaderPatcherCreateVertexProgram(g_patcher, id, g_attribs,
                                       g_attrib_count, g_streams, g_stream_count, &vp);

SceGxmFragmentProgram* fp;
sceGxmShaderPatcherCreateFragmentProgram(g_patcher, id,
    SCE_GXM_OUTPUT_REGISTER_FORMAT_UCHAR4, SCE_GXM_MULTISAMPLE_NONE,
    NULL, vertex_program, &fp);
```

Two consequences worth internalising:

- fragment programs are created **per vertex program and per blend state** — this is
  the source of the permutation explosion
- everything you register consumes shader-patcher cache memory; size that buffer for
  the number of distinct programs you expect, not the number you have today

## 6. When to compile at runtime and when not to

| Situation | Choose |
|-----------|--------|
| fixed shader set (post-FX chain, UI, skybox) | **offline `psp2cgc`** — faster, no device dependency |
| shader keyed by GPU state (GX/TEV, console HLE) | runtime compile + persistent cache |
| GL game through vitaGL | runtime, automatically |

For the runtime path, cache aggressively: hash the state key → RAM table → writable
cache file → packaged cache → compile on a true miss. The `size` and program bytes
can be written straight out as a `.gxp`, which is exactly what the sample above
does — that file is loadable next boot with no compiler involved.

## 7. Device prerequisites — say them in your README

| Requirement | Consequence if missing |
|-------------|------------------------|
| `ur0:data/libshacccg.suprx` | every compile returns an error; black screen |
| `SceShaccCgExt` installed (if using `shark_init`) | extensions missing, compile fails |
| `libvitashark`, `SceShaccCg_stub` in the link line | unresolved symbols at link |
| `SceShaccCgExt` before `vitaShaRK` at install time | version-skew garbage programs |

Never bundle `libshacccg.suprx`. The user extracts it from their own console.

