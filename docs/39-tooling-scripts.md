# 39 — Upstream Tooling Worth Borrowing

Four small scripts and three templates that catch the mistakes everybody makes. They
are **not vendored in this repository** — fetch them from
[yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill) (MIT) under
`skills/vita-port/scripts/` and `skills/vita-port/assets/`.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (MIT). See [CREDITS](../CREDITS.md).

---

## 1. `check_env.sh` — is this machine a usable VitaSDK host?

```bash
./check_env.sh            # base checks
./check_env.sh vitaGL taihen   # also check vdpm packages
```

Checks `VITASDK` (and discovers `~/vitasdk`, `/usr/local/vitasdk`),
`arm-vita-eabi-gcc`, `vita-elf-create`, `vita-make-fself`, `vita-mksofex`,
`vita-pack-vpk`, `share/vita.toolchain.cmake`, `share/vita.cmake`, `$VITASDK/bin` on
`PATH`, plus `cmake`, `make`, `git`, `python3`. Prints `[ok]` / `[MISS]` and exits
non-zero if anything is missing.

**This is the fastest way to catch the "extracted SDK" problem** from doc 02: a tree
with `include/` and `lib/` but no `share/` or `bin/` fails here loudly instead of
failing inside CMake with a confusing toolchain error.

## 2. `check_vpk.py` — will this package install?

```bash
python3 check_vpk.py build/mygame.vpk
```

Verifies:

| Check | Failure it catches |
|-------|--------------------|
| `eboot.bin` starts with `"SCE\0"` | installs, exits immediately |
| `param.sfo` parses | install fails silently |
| `TITLE_ID` is exactly 9 characters | bubble missing |
| `APP_VER` matches `NN.NN` | metadata rejected |
| `icon0.png` 128×128, **indexed colour**, no alpha | blank bubble |
| `bg0.png` 840×500, `startup.png` 280×158 | stretched / missing art |
| `template.xml` well-formed | broken LiveArea page |

Exit 0 = no errors (warnings allowed), 1 = errors. This directly targets
`0x8010113D` and the "installs but no bubble" class in doc 34.

## 3. `make_placeholder_assets.py` — valid art on day one

Generates correct-size, indexed-colour LiveArea assets so a project can produce an
installable VPK before any artist is involved. Use it when `check_vpk.py` complains
about PNG colour type or dimensions.

## 4. `audit_source.py` — does this codebase fit the Vita?

```bash
python3 audit_source.py path/to/src --max-hits 50
python3 audit_source.py path/to/src --json > audit.json
```

Greps for constructs that do not map onto the platform: `fork`/`exec`/`mmap`/`popen`,
dynamic loading and JIT, threads, AWT/ImageIO, NIO channels, JNI, `gdx.gl30`. Each
hit carries a severity and the usual remedy, and it prints a feasibility hint.

**It is a grep, not a compiler.** Expect false positives from comments and
`#ifdef`-ed-out code. Use it to build a work list, then read the hits (doc 36).

## 5. `CMakeLists.vita.txt` — a correct CMake skeleton

Worth reading even if you write your own:

- `CMAKE_TOOLCHAIN_FILE` resolved from `VITASDK` **before** `project()`
- `include(${VITASDK}/share/vita.cmake)` **after** it
- the three `VITA_*` values at the top, with the 9-character title-ID rule in a comment
- `-std=gnu11` and `-fsigned-char` for old code, `-g` kept for crash dumps
- `ATTRIBUTE2=12` explained, including that released SDKs otherwise write `0`
- one place to pick a graphics route, with dependents listed before dependencies

## 6. `vita_runtime.c` — the four globals everyone forgets

```c
unsigned int _newlib_heap_size_user = 192 * 1024 * 1024;
unsigned int sceLibcHeapSize = 4 * 1024 * 1024;
const unsigned int sceUserMainThreadStackSize = 8 * 1024 * 1024;
const int sceUserMainThreadPriority = 0x60;
```

plus `ensure_data_dir()` (`app0:` is read-only, so `ux0:data/<app>` must be created)
and `set_clocks()`. It also has a `vita_pin_thread_to_core()` helper with affinity
masks `0x10000 / 0x20000 / 0x40000` verified in a shipping port.

Why it matters: `_newlib_heap_size_user` is taken as a **single memblock at
startup**. If it exceeds free MAIN memory, newlib silently ends up with an empty heap
and every `malloc` returns ENOMEM — which presents as "crashes on boot with no log".

## 7. `template.xml` — minimal valid LiveArea

Background image, startup image, gate. Small enough to read in one screen, correct
enough to satisfy the parser.
