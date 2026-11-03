# 42 — A Complete Direct-GXM Sample

`gxmfun` is the smallest *complete* GXM program in this workspace: no engine, no
`vita2d`, no `vitaGL`. A cube, a camera, a portal effect, a debug overlay and correct
frame synchronisation.

> Source: `Other Emulators/gxmfun-master` (12 KB of C + 8 Cg shaders).
> See [CREDITS](../CREDITS.md).

---

## 1. Why it is worth reading

Everything in [doc 04](04-gxm-rendering.md) in working order, at a size you can read
in one sitting:

```
CMakeLists.txt      psp2cgc → .gxp → objcopy → link   (the whole shader pipeline)
shader/*.cg         8 shaders: clear, color, cube, disable-color-buffer
include/math_utils.h vector/matrix types (12 KB of implementation)
include/camera.h    a first-person camera
source/main.c       55 KB — GXM setup, scene, draws, frame sync
```

## 2. The display configuration, up front

```c
#define DISPLAY_WIDTH        960
#define DISPLAY_HEIGHT       544
#define DISPLAY_STRIDE       1024          /* NOT 960 */
#define DISPLAY_BUFFER_COUNT 2
#define DISPLAY_COLOR_FORMAT   SCE_GXM_COLOR_FORMAT_A8B8G8R8
#define DISPLAY_PIXEL_FORMAT   SCE_DISPLAY_PIXELFORMAT_A8B8G8R8
#define MAX_PENDING_SWAPS     (DISPLAY_BUFFER_COUNT - 1)
```

Two details that trip people:

- **stride is 1024, not 960.** GXM surfaces are aligned; using the visible width as
  the stride gives a sheared image.
- **`MAX_PENDING_SWAPS = buffers - 1`.** With two buffers you may have exactly one
  frame in flight. Pushing more than that is a race the display queue will not save
  you from.

## 3. Embedded shaders

`psp2cgc` compiles each `.cg`, then `objcopy -I binary` turns it into an object file.
The linker exposes `_binary_<name>_start`, so the program is just a symbol:

```c
static const SceGxmProgram* const g_program_cube_v =
    (SceGxmProgram*) &_binary_cube_v_gxp_start;
static const SceGxmProgram* const g_program_cube_f =
    (SceGxmProgram*) &_binary_cube_f_gxp_start;
```

No file IO, no runtime compiler, no `libshacccg` dependency. This is the pattern to
copy for any fixed shader set.

## 4. Shader patcher objects, named per program

```c
static SceGxmShaderPatcherId g_cube_vertex_program_id;
static SceGxmShaderPatcherId g_cube_fragment_program_id;
static const SceGxmProgramParameter* g_cube_v_position_param;
static const SceGxmProgramParameter* g_cube_v_normal_param;
static const SceGxmProgramParameter* g_cube_v_color_param;
static const SceGxmProgramParameter* g_cube_v_mvp_param;
static const SceGxmProgramParameter* g_cube_f_modelview_param;
static const SceGxmProgramParameter* g_cube_f_normal_matrix_param;
```

Resolve every parameter **once at startup** and keep the pointer. The names are
verbose on purpose — when you have eight programs, `g_cube_f_normal_matrix_param` is
the difference between a five-second fix and a twenty-minute hunt.

## 5. Vertex layouts as structs

```c
struct color_vertex  { vector3f position; vector4f color; };
struct mesh_vertex   { vector3f position; vector3f normal; vector4f color; };
```

One struct per stream configuration, used for both the CPU-side array and the
`SceGxmVertexAttribute` table. Keeping them in one place is what stops the classic
"attribute offsets drifted from the struct" bug.

## 6. Frame structure

1. begin scene on the current colour surface + depth/stencil surface
2. clear (a dedicated two-triangle "clear" shader writes the clear colour — GXM has
   no `glClear`)
3. draw the cube with its matrix uniforms
4. draw the portal / second pass with the colour buffer disabled where needed
5. end scene, hand the finished surface to the display queue
6. rotate to the next buffer, respecting `MAX_PENDING_SWAPS`

The `disable_color_buffer` shader pair exists purely to do a depth-only pass — a
small example of "if GXM does not have the fixed-function feature, write a shader".

## 7. Camera and math

`camera.c` + `math_utils.c` provide vectors, matrices, perspective/look-at and the
camera state. 12 KB of straightforward C with no dependencies — a decent starting
point for any project that does not want to pull in GLM.

## 8. What to take from it

| Take | Why |
|------|-----|
| `objcopy`-embedded `.gxp` symbols | zero runtime shader dependency |
| stride 1024, `MAX_PENDING_SWAPS` discipline | correct display timing |
| clear-as-a-shader | GXM has no fixed-function clear |
| one struct per vertex layout | attribute tables stay honest |
| per-program parameter pointers resolved once | no per-draw lookups |
| depth-only pass via a shader | how to get effects GXM lacks |

If you are writing your own renderer and have never touched GXM, read this file
before reading the 130 KB Melee renderer.
