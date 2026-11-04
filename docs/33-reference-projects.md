# 33 — Reference Projects Catalogue

Real, published Vita ports grouped by route. Use this to answer "has someone done
this already" — forking beats starting over, and reading a shipped port beats
guessing.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/port-types.md`, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Route → example

| Route | Representative project | What it proves |
|-------|------------------------|----------------|
| Source/decomp + GXM | Melee Vita (`zm2283145/melee-Vita`) | a full 3D fighter, GX→GXM, shader cache |
| Source/decomp + vitaGL | ACGC (`Brendonm17/ACGC-Vita-Port`) | 161 plain-C `GX*` functions + 52 precompiled shaders |
| Native recompile + vitaGL | THUG (`Tony Hawk's Underground`) | recompiled game with a declared source platform |
| Native recompile + ELF loader | Source engine / HL2 | custom module loader, ToGLES dialect |
| Android `.so` loader | VitaKotor2 (KOTOR II) | multi-`.so` fixed-address maps, bigalloc isolation |
| Android `.so` loader | Tomb Raider 1 & 2 (`raider-vita`) | a complete loader with trophies |
| Engine runtime loader | YoYo Loader (GameMaker) | one runtime, whole catalogue of titles |
| Cross-platform HAL | mkwiita (MKW for 3DS → Vita) | ~2,200 lines of layer, game unchanged |
| Static recompilation | Conker (`CBFD-Recompiled`), Isaac (`repentogxm`), MH3 (`mh3-wii`) | lift → C → native |
| Emulation, dynarec + vitaGL | DaedalusX64, PCSX AwO | ARM codegen (97 KB generator), HLE graphics |
| Emulation, interpreter + vita2d | melonDS | software raster + blit, no shaders at all |
| Source port, id Tech | vitaQuakeII, vitaWolfen | a `psp2/` platform directory is the whole port |
| Homebrew engine | Xenity Engine | one core, four platforms, thin per-platform layer |
| Interpreter / data-driven | EasyRPG Player, GemRB, fake-08, Aleph One, FLARE | engine ships, user brings content |
| Scripting runtime | CPython 2.7 port, VitaMonoLoader | ship the language, not the game |
| Engine as a shared library | Godot 4.8-dev Vita port | `libgodot.so` + a small import-resolving loader |
| Engine content patch | Dark Queen of Mortholme | PCK repack around an existing runtime |

## 2. Projects whose *patterns* are worth reading

| Project | Pattern |
|---------|---------|
| Melee Vita | shader permutation cache, resolution scaling with native fallbacks, disc streaming thread |
| VitaKotor2 | big-allocation isolation (`bigalloc.c`), whole-process core pinning, GL worker thread, SD-card I/O batching |
| DaedalusX64 | dynarec **without** kubridge, `CIRCULAR_POOL_SPEEDHACK`, zero-copy vertex/index lists |
| ACGC | precompiled shader bank + uber shader; plain-C HLE surface |
| encounter/aurora | what a complete GX HLE must cover (GX, GD, OS, PAD, SI, VI, DVD, CARD, AR, THP); SQLite+zstd pipeline cache |
| THUG | 2003-era codebase compat header, per-file `-O2`, translated-path probe file |
| HL2 / Source | multi-pool memory, kernel exec mapping, data-driven cvars |
| mkwiita | priority/affinity mapping, lazy kernel objects, quality.txt switches |
| solobop / soloader-template | the canonical Android loader skeleton |

## 3. Where to search before committing

- GitHub topics: `psvita-homebrew`, `vitasdk`
- plain searches: `"<game> vita"`, `"<game> psvita"`, `"<game> recomp"`, `"<game> decomp"`
- r/vitahacks, r/vitapiracy release threads (Reddit's JSON API is bot-blocked;
  use archive mirrors or web search)
- VitaDB — a JS single-page app, so check it in a browser, not with curl

There is no single authoritative masterlist. Cast wide, then verify the port actually
runs (see doc 27 §7 on believing status claims).

## 4. Portable-codebase trick

Before writing a from-scratch port or a VM workaround, look for an **existing C/C++
or platform-agnostic reimplementation** of the same thing: a core separated from UI,
a GPL engine rewrite, a decomp/recomp tree, a translation of the format/save logic.

Model: PKSM (FlagBrew) — a Pokémon save manager whose save logic lives in a portable
`core/` plus a shared `common/` layer, with only the 3DS front-end platform-specific.
Porting the core is a week; porting the app is a month.

## 5. Closed-source commercial games

| Situation | Honest answer |
|-----------|---------------|
| engine has an open reimplementation that reads the user's data | viable (framework route) |
| a lift/recomp exists for that platform | viable (static recompilation) |
| Android build exists and is ARMv7 | viable (`.so` loader) |
| none of the above | not viable — say so early |

Never distribute the game's data, its `.so`, or a retail binary you did not build.
