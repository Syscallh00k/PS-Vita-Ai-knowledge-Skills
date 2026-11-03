# 38 — Java / libGDX Games (TeaVM C Backend)

The Vita has no JVM. The working route is ahead-of-time translation: TeaVM's C
backend turns reachable Java bytecode into C11, a hand-written libGDX backend
replaces the desktop one, and VitaSDK compiles the result into one static ELF.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/java-teavm.md`, MIT). Reference implementation: STSPSVITA (MIT).
> See [CREDITS](../CREDITS.md).

---

## 1. When this route fits

| Fits | Does not fit without a large rewrite |
|------|--------------------------------------|
| libGDX on GL 2.0 / GLES2, 2D or light 3D | GL 3.x (`Gdx.gl30`, GLSL 330+, texture arrays, integer textures) |
| single-threaded game logic (async helpers become fibers) | real parallelism needed for performance |
| reflection over classes known at build time | reflection over classes chosen from data at runtime (plugin systems, generic deserialisers) |
| plain `java.io` file access | AWT/Swing/ImageIO, `FileChannel`/mmap, JNI libs without a C equivalent |
| Java heap + textures within ~365 MiB | large 3D worlds whose live heap alone nears 200 MiB |

Any native libGDX extension the game uses (FreeType, Box2D, Bullet, controllers) must
be rebuilt for Vita and bound through TeaVM's `@Import` instead of JNI. Scan the jar
or source before committing — `audit_source.py` (upstream, see doc 39) flags AWT, ImageIO,
reflection, threads, NIO channels, JNI and `gdx.gl30` in `.java` files.

## 2. Choosing between the AOT routes

Name the route you picked **and** why it beat the others, because the trade-offs
differ on licence, threads, and how much backend already exists.

| | TeaVM C backend | Clearwing VM | Native rewrite |
|---|---|---|---|
| Output | C11 | C++20 | your own C/C++ |
| Licence of tool/runtime | Apache-2.0 (backends vary) | GPL-2.0 — check for a linking exception before shipping non-GPL code | yours |
| Vita precedent | STSPSVITA (Slay the Spire, pre-alpha) | none published; targets are 64-bit, so 32-bit ARM is untested | many C/C++ ports |
| libGDX backend | GL 2.0 only, written against libGDX 1.9.5 | mature (several shipped Switch titles), no GL30 | reimplement what you use |
| Threads | cooperative fibers on one OS thread | real threads | real pthreads |
| Reflection | full in TeaVM 0.16 if kept by the reflection policy | partial | n/a |

## 3. Architecture

```
 Java / libGDX game
      │  TeaVM C backend (AOT)
      ▼
 generated C11  ──┬── runtime patches (allocator, fibers, string, NIO shims)
                  ├── hand-written libGDX backend (GL 2.0 → vitaGL)
                  └── Java class library subset
      │
      ▼  VitaSDK + vitaGL
 one static ARM ELF → SELF → VPK
```

## 4. Class library gaps you will hit

TeaVM implements the reachable subset of the JDK. Expect to patch or shim:

| Area | Symptom |
|------|---------|
| `java.io` file APIs | paths resolve against the wrong root — map to `ux0:data/<app>/` |
| `String` internals | patched `intern` behaviour in older TeaVM |
| NIO buffers/channels | non-conformant in older versions; avoid or shim |
| reflection | works only for classes the reflection policy keeps; add explicit registrations |
| `RandomAccessFile` | patched in newer TeaVM |

**Version matters**: pin the TeaVM version and re-check every runtime patch anchor
when you move it. Patches written against 0.15 output do not necessarily apply to
0.16 output.

## 5. Runtime patches

The generated C needs help that a desktop host build hides:

- allocator: the generated code assumes a large flat heap; back it with one
  memblock and a bump/free-list allocator rather than thousands of `malloc` calls
- fibers/coroutines: TeaVM's cooperative yields must map onto one OS thread — do not
  map them to pthreads
- string and NIO shims, as above

## 6. libGDX backend

The backend is the real work: window/input, textures, shader program binding, sprite
batcher, file handle, audio stubs. STSPSVITA's backend is GL 2.0 only and written
against libGDX 1.9.5 — read it as a template, not a library.

Textures should be re-encoded offline to a native vitaGL format (doc 24). The atlas
format does not need to change.

## 7. Native side: memory and startup

Treat it as any other Vita app:

- `_newlib_heap_size_user` sized for the Java heap + texture budget
- `ATTRIBUTE2=12` in `param.sfo` for the larger memory budget
- one thread runs the game loop; the render calls stay on the GL init thread
- `libshacccg.suprx` required at `ur0:data/libshacccg.suprx` (vitaGL path)

## 8. Assets and legality

Assets come from the user's own copy — a jar/OBB they own, or files they extract
themselves. Ship the pipeline and the code, not the data. Same rule as every other
route in this repository (doc 27).

## 9. Effort

Honest scale for a 2D libGDX game with no exotic dependencies: **weeks to months**,
dominated by the class-library gaps and the libGDX backend, not by the translation
itself. If the game needs GL 3.x or heavy runtime reflection, the effort multiplies
and the route should be reconsidered before starting.
