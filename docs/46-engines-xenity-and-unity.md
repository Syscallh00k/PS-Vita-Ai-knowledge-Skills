# 46 — Homebrew Engines (Xenity) and Unity on Vita

Two "engine-first" routes: an engine written for the Vita from the start, and the
commercial engine with an official-but-constrained Vita target.

> Sources: `Other Emulators/Xenity-Engine-main`,
> `Other Emulators/PSVita-Unity-Utilities-main`. See [CREDITS](../CREDITS.md).

---

## 1. Xenity Engine — a cross-platform homebrew engine

A GameObject/Component engine targeting **Windows, PSP, PS Vita and PS3** from one
codebase. Features (per its README): basic 2D and 3D, 2D audio, skyboxes, scenes,
3D physics, with animations and 2D physics still open.

### Layout

```
Xenity_Engine/
  Source/
    engine/     shared runtime (scenes, components, mesh, materials)
    psvita/     the Vita platform layer
    psp/  ps3/  ps2/  windows/  editor/  unit_tests/
```

The Vita platform layer is deliberately small — `psvita/cpu.cpp`,
`psvita/inputs/inputs.cpp`, `psvita/debug/debug.cpp`, plus a bundled
`common/debugScreen.c` (37 KB, a full on-screen debug renderer with font).
Everything else is shared code that compiles for four platforms.

### What that proves

- a *shared* engine core with a thin platform layer is viable on this hardware
- the per-platform layer can be hundreds of lines, not thousands
- an on-screen debug renderer is worth bundling — they ship one, so do Melee, so
  does PrincessLog

### What it costs

It is a hobby-scale engine: no animations, no 2D physics yet. But if your target is
"a 3D-ish game on Vita", starting from a working cross-platform engine is cheaper
than building a renderer first.

## 2. Unity on Vita — the constrained route

There **was** an official Unity Vita target, on Unity 2018-era versions, PC-hosted.
Two things follow:

1. the output is a Unity player binary — not something you can build from modern
   Unity (2020+) without the vendor target
2. porting a *modern* Unity game therefore means shipping a runtime for its managed
   assemblies, which is the fragile route

### The tooling that exists

`PSVita Unity Utilities` is a Unity **editor** extension (2018.2.19f1-era) that
handles the annoying parts:

| Feature | What it saves |
|---------|---------------|
| VPK building | no manual `vita-make-fself`/`vita-pack-vpk` |
| FTP transferring | push builds to the console without a cable |
| build & run (WIP) | one button |
| Vita control | reboot/launch from the editor |
| TitleID / project settings access | no hunting through project files |
| **TitleID check against VitaDB** | catches a duplicate ID *before* installing |

That last one is a genuinely good idea for any homebrew project: check your chosen
title ID against the public database before you ship, because duplicates break
installs in ways that look like packaging bugs.

Practical requirements from its README:

- Scripting Runtime Version: **.NET 4.x Equivalent**
- Build Settings: **PSVita, PC Hosted**
- console power-save: standby 10–30 min, WiFi-in-standby off — otherwise FTP pushes
  die halfway

## 3. Choosing between the three "engine" routes

| Route | You get | You pay |
|-------|---------|---------|
| Homebrew engine (Xenity) | full control, small footprint, proven renderer | limited features, you maintain the engine |
| Commercial engine runtime (Unity/GameMaker loaders) | game logic free | fixed capabilities, licence constraints, per-game compatibility work |
| Your own renderer | exactly what you need | months before the first frame |

Decision rule: if the game's logic is the hard part, use a runtime/loader. If the
*game* is simple and the *look* is the hard part, use or build a small engine.

## 4. Title IDs, one more time

Nine characters, `XXXXYYYYY`, unique on the device **and** in the public database.
Both a utilities tool and a manual check are worth having:

```
check_vpk.py           validates length/format locally
VitaDB title-ID check  validates uniqueness publicly
```

A duplicate ID does not fail loudly; it silently replaces or refuses, and you debug
the wrong thing for an hour.

## 5. Source-available engine caveat

Every engine in this section is either GPL-ish or explicitly MIT-ish, and all of them
say "expect this to change". Pin the commit you built against, and record its licence
in your own README with a link to the upstream file.
