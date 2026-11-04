# 51 — Scripting Runtimes on the Vita

If the game's logic is Python or C#, you can ship the runtime instead of porting the
game. Both have working precedents here.

> Sources: `new imports/cpython-vita-2.7` (CPython 2.7.18 port),
> `new imports/VitaMonoLoader-main` (Mono AOT loader).
> See [CREDITS](../CREDITS.md).

---

## 1. CPython 2.7 — the autotools recipe

`build_vita.sh` is short and worth keeping verbatim in spirit:

```sh
./configure --build=armv7 --host=arm-vita-eabi \
  --prefix="$VITASDK/arm-vita-eabi" \
  --enable-optimizations --disable-shared --with-threads \
  --disable-ipv6 --without-signal-module \
  CFLAGS="-Wl,-q,-z,nocopyreloc" \
  CPPFLAGS="-Wl,-q,-z,nocopyreloc" \
  LDFLAGS="-Wl,--allow-multiple-definition"
make -j$(nproc) install
```

Each flag earns its place:

| Flag | Why |
|------|-----|
| `--host=arm-vita-eabi` | cross-compile; `--build=armv7` avoids a config triple fight |
| `--disable-shared` | no `libpython.so` — everything static |
| `--with-threads` | newlib pthread path |
| `--disable-ipv6`, `--without-signal-module` | Vita has neither in a usable form |
| `-Wl,-q` | keep relocations (the standard Vita requirement) |
| `-z nocopyreloc` | avoids text relocations the Vita loader rejects |
| `--allow-multiple-definition` | newlib/SDK symbol overlap |

That combination is the difference between "Python builds" and "Python links".

## 2. Shipping the standard library

A full Python install is tens of thousands of files. The port shrinks it to a single
zip:

```sh
rm -rf test lib2to3/tests      # never needed at runtime
find . -type l -not -name "*.py" -delete
python2 -OO -m compileall .
find . -type l -not -name "*.pyc" -delete
7z a -l ../python27.zip .
```

Result: one `python27.zip` beside the binary. **One archive instead of 10k files** —
the same lesson as doc 24 §7, and it makes startup measurable rather than awful.

## 3. What a shipped scripting runtime costs

| Cost | Reality |
|------|---------|
| memory | interpreter + stdlib zip + your game's live objects |
| startup | seconds, not milliseconds — show something on screen |
| determinism | `-O` bytecode and float behaviour differ from desktop; test gameplay |
| debugging | you now debug two programs (host and script) |
| security model | there is none; scripts are as trusted as your binary |

## 4. Mono / .NET — the AOT route

`VitaMonoLoader` shows how C# reaches the Vita: **ahead-of-time compile the managed
DLL to ARM assembly**, then link it in.

Pipeline:

```
C# source  ──mcs──▶  MyDll.dll  ──mono-xcompiler──▶  MyDll.dll.s  ──gcc──▶  app
                      (managed)      --aot=full,asmonly,nodebug,static
```

Activation on device:

```c
extern void** mono_aot_module_MyDll_info;   /* emitted by the AOT assembly */
VMLRegisterAssembly(mono_aot_module_MyDll_info);
```

Hard requirements, from the project's own instructions:

1. **the PSP2 application must be compiled in ARM mode** (`-marm` with vitaSDK,
   `-Xthumb=0` with the SCE SDK) — the AOT stubs are not Thumb-compatible
2. **CapUnlocker plugin** installed (memory permissions)
3. Mono support files copied to `app0:Media/` — `Managed/mono/2.0/machine.config` and
   the runtime modules
4. Mono headers added to the SDK for the host-side build
5. host tooling: Mono for Windows + the Unity PS Vita support package (for
   `mono-xcompiler.exe`)

Note what this implies: this route is really **"run a Unity Vita build you generate"**,
not "run an arbitrary Unity game". The Unity Vita editor support is 2018-era, and the
AOT step is per-DLL.

## 5. Lua and other embedded interpreters

Aleph One ships Lua for scripting, and it needs nothing special on Vita — Lua compiles
cleanly with newlib and is happy in a few hundred KB. If a game's logic is already
Lua, that is the easiest scripting port you will ever do: embed the interpreter,
provide the host API, ship.

## 6. Choosing between them

| Situation | Route |
|-----------|-------|
| Game logic is Python and the game is small | ship CPython (doc here) |
| Game is a Unity title with an Android/PSV build | loader (doc 19) or Mono AOT |
| Game logic is Lua | embed Lua, no runtime port needed |
| Game logic is native code | none of the above — see docs 20/26/30 |

## 7. The rule that applies to all of them

**A scripting runtime on this hardware is a memory decision, not a language decision.**
Budget interpreter + stdlib + your data before you write a line, and decide what gets
dropped when it does not fit. Shipping the runtime is easy; shipping the *content* is
what runs out of room.
