# 50 — Interpreter and Data-Driven Engines

Not every port needs an engine rewrite. Six projects in this workspace run *content*
through an interpreter, and that shape is the cheapest route of all.

> Sources: `new imports/Player-master` (EasyRPG), `new imports/gemrb-vita-master`,
> `new imports/fake-08-master`, `new imports/Aleph-One-PsVita-main`,
> `new imports/flare-engine-master`, `new imports/VITA-8-sw-render`.
> See [CREDITS](../CREDITS.md).

---

## 1. The family

| Project | What it runs | Engine model |
|---------|--------------|--------------|
| **EasyRPG Player** | RPG Maker 2000/2003 games | reads LCF data (`liblcf`), interprets events |
| **GemRB** | Infinity Engine games (BG1/2, IWD, PST) | reimplements the engine, reads original data |
| **fake-08** | PICO-8 carts | implements the fantasy console (Lua subset + API) |
| **Aleph One** | Marathon + scenarios | open-source engine, data-driven scenarios |
| **FLARE** | its own RPGs | open engine, data-defined tilesets/items/maps |
| **VITA-8** | CHIP-8 ROMs | 200-line interpreter |

Every one of them ships the engine and *never* the content. The user brings the game
data. That is why they exist on this platform at all.

## 2. What each one needs from the platform

| Project | Graphics | Audio | Notes |
|---------|----------|-------|-------|
| EasyRPG | SDL2 (+ pixman, libpng, zlib) | SDL2 mixer / mpg123 / WildMIDI | very portable C++; SDL2 abstract target |
| GemRB | SDL 1.2 (`Northfear/SDL-1.2-vita`) | SDL_mixer | also needs `libpython`; per-game `.cfg` |
| fake-08 | SDL2 (`platform/vita/source/VitaHost.cpp`) | SDL2 | host layer is one file |
| Aleph One | SDL2 | SDL2 | `vita_init.c`, `-D__vita__` |
| FLARE | SDL2 + vitaGL | SDL2 mixer | `Makefile.vita` |
| VITA-8 | raw framebuffer | none | historical (psp2sdk era) |

The pattern is blunt: **if the engine already speaks SDL2, the port is a host layer
and a Makefile.** Three of the six are literally that.

## 3. GemRB's lesson: one runtime, many games

GemRB ships a **LiveArea launcher that picks a game**, and each game gets a config
file and a default data folder:

```
ux0:data/GemRB/BG1.cfg   → GamePath=ux0:data/GemRB/BG1
ux0:data/GemRB/BG2.cfg   → ...
ux0:data/GemRB/IWD.cfg   → ...
```

Practical notes worth stealing:

- **auto-detection of the game type does not work** — set `GameType` manually in the
  config rather than trusting a detector
- **suspend/resume needs `rePatch` or `FdFix`** (one only, never both) — without a
  fd fix, a resumed process holds dead descriptors
- the LiveArea can launch different titles into different configs, which is how you
  ship "five games" as one bubble

That config-per-title layout is the same idea as PCSX AwO's per-game settings (doc
21 §7), and it is the right answer for anything that hosts multiple data sets.

## 4. fake-08's lesson: the host layer is one file

`platform/vita/source/VitaHost.cpp` (9.5 KB) plus a Makefile is the entire Vita port
of a PICO-8 player. The engine, the Lua interpreter, the cart loader — all shared
with 3DS, Switch, Wii U, Miyoo.

This is what a well-factored engine gives you, and it is the argument for choosing
projects that already have a `platform/` directory: your work is the directory.

## 5. Aleph One's lesson: a scenario loader is a feature, not an afterthought

Aleph One runs Marathon *and* a large library of fan scenarios. The port's install
instructions are essentially "put each game in its own folder under
`ux0:data/AlephOne/`". The engine lists them at boot.

Consequence: the port never needs to know which game it is running, and adding a
scenario costs the user nothing. If your target has modular content, design the
folder layout first — it removes a whole class of "which version?" support questions.

## 6. When to prefer this route

| Condition | Why it wins |
|-----------|-------------|
| Engine is open source and speaks SDL2 | port is days |
| Content is user-supplied by design | no legal negotiation, no asset conversion |
| Game logic is data, not compiled code | you never reimplement gameplay |
| A `platform/` seam already exists | your diff is small and mergeable |

And when not: if the engine has no seam, or the content is compiled/binary (see docs
19, 26, 30), this route does not exist.

## 7. The transferable checklist

- [ ] does the engine already have a `platform/` directory?
- [ ] does it use SDL2 (or 1.2 with a Vita fork)?
- [ ] is the content read at runtime from user-supplied files?
- [ ] is there a config file per game/data set?
- [ ] can the LiveArea launch different configurations?
- [ ] does it need `FdFix`/`rePatch` for suspend/resume?
- [ ] can a second data set be added without a rebuild?
