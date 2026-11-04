# 47 — Shipping Your Engine as a Shared Library

A pattern almost nobody uses, and it solves a real problem: build the engine as a
PIC `.so`, and let a small loader own the process.

> Sources: `new imports/godot4-psvita-main/platform/vita/` (Godot 4.8-dev Vita port),
> `new imports/VitaMonoLoader-main`. See [CREDITS](../CREDITS.md).

---

## 1. The layout

```
app.vpk
├── eboot.bin            small loader (main.c + so_util + default_dynlib)
├── libgodot.so          the whole engine, PIC, hard-float ARM
└── Media/, tests/       assets and the test project
```

`eboot.bin` does what an Android loader does, except the module it loads is **its own
engine** rather than someone else's game:

1. `so_file_load()` the `.so`
2. resolve every import against `default_dynlib.c` (62 KB of name→function table)
3. patch what must be patched
4. flush caches, run init, hand over

## 2. Why bother

| Problem | How the `.so` approach fixes it |
|---------|--------------------------------|
| Rebuild the loader for every engine change | the loader is ~25 KB and almost never changes |
| Import surface is huge and grows | imports are validated **at package time** against the dynlib table, not discovered at runtime |
| Debugging symbols bloat the shipped binary | ship a `strip --strip-unneeded` copy, keep the full `.so` for GDB |
| Iteration | FTP a new `.so` (~tens of MB) instead of rebuilding the whole app |
| Multiple games, one runtime | swap the asset set, keep the `.so` |

The build even encodes the split: `vita_shared=yes` produces `bin/libgodot.so` with
static engine objects linked in, leaving external libraries as unresolved imports for
the loader.

## 3. What you have to build that SCons/CMake won't give you

| File | Purpose |
|------|---------|
| `so_util.c` | ELF load/relocate/resolve (the same code Android loaders use) |
| `default_dynlib.c` | the import table — one row per external symbol |
| `tls_stubs.c` | thread-local storage shims; the engine assumes TLS the loader must provide |
| `dl_stubs.c` | `dlopen`/`dlsym`/`dlclose` stubs |
| `sdk_shims.c` | small libc/SDK differences |
| `boot_splash.c` | something on screen while the `.so` loads (a black screen this early reads as a crash) |

TLS is the part people miss. A C++ engine uses `thread_local` extensively; newlib's
TLS support through this path needs explicit help, and without `tls_stubs.c` you get
instant corruption rather than a clean error.

## 4. Packaging discipline that actually prevents bugs

The Godot port's packager (`tools/package_runtime.py`) does four things worth copying:

1. **verifies the ELF** — format, relocation types, entry point
2. **verifies every import** against `default_dynlib.c`; a missing one aborts the
   package with the symbol name, instead of crashing on device
3. **strips the shipped copy** and re-validates the stripped ELF; keeps the
   unstripped original for debugging, with its path+hash in a manifest
4. **records hashes** of runtime, VitaGL, loader and scene into a manifest the loader
   prints at boot

That last one is the cheapest debugging tool on this platform: the loader's first
lines tell you exactly which build is running.

## 5. The GXM bridge option

The same port also demonstrates a second renderer path: instead of translating
GLES3, it routes 2D canvas commands into **`libvita2d`'s prebuilt GXM shaders**
(`gxm_bridge.c`, 40 KB). No GLES3 shader translation, no `psp2cgc`.

Consequences worth knowing:

- works because `libvita2d` ships embedded GXM programs — usable even when the
  offline Cg compiler is unavailable
- limited to what vita2d can express: rects, atlas regions, flips, nearest/linear
  filtering; no mipmaps, no MSDF fonts, no custom materials
- uploads must respect the **real GXM stride**, not the logical width
- a texture budget of 48 MiB in 256 KiB pages, with no LRU — an over-budget
  allocation is refused with a diagnostic rather than evicting

This is the pragmatic middle ground between "vitaGL everything" and "write a full
GXM renderer".

## 6. When this pattern is right for us

| Situation | Verdict |
|-----------|---------|
| Engine is big, changes often, and has a huge libc surface | good fit |
| You need to swap content without touching code | good fit |
| You want the engine debuggable with symbols on device | good fit |
| Tiny app, few imports | overkill — just link statically |

For a project like ours, the appeal is concrete: our runtime will grow for months,
and a stable loader means the thing that boots never has to be re-validated.

## 7. Costs

- one more thing to keep in sync (`.so` ↔ dynlib table)
- import failures move from link time to package time — only a win if you actually
  run the verifier
- a `.so` that is 40 MB must be read fully into memory before segments are mapped;
  hence the strip step in the pipeline
