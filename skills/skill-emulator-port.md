---
name: emulator-port
description: Port an emulator to the Vita — decide dynarec vs interpreter, size memory, pick the renderer, and build the frontend people actually need.
when: The target is an emulator, or a game whose only route is emulation.
---

# Skill: Emulator Port

## Step 0 — Is emulation even viable

| Guest | On a 444 MHz A9 |
|-------|-----------------|
| PS1 | playable at 60 fps with a good renderer |
| N64 | borderline; needs dynarec and frame skipping |
| DS | workable — melonDS runs with an *interpreter* + software raster |
| PSP / PS2 / anything newer | not viable |

If the guest is newer than the host by a wide margin, stop and re-triage
(doc 30 static recompilation is the realistic alternative for closed consoles).

## Step 1 — Find the platform seam

Look for an existing per-OS directory in the emulator (`SysVita` equivalents:
`sys_*.c`, `vid_*.c`, `snddma_*.c`, `in_*.c`). If it exists, your port is that
directory. If the emulator has no platform seam, budget a refactor first.

**Gate:** you can name the files you will write. Aim for a dozen.

## Step 2 — Dynarec or interpreter

| Choose dynarec when | Choose interpreter when |
|---------------------|-------------------------|
| guest is PS1/N64-class and must hit 60 fps | guest is DS-class ARM at moderate clocks |
| an ARM dynarec already exists (PSP ports) | no ARM backend exists and you cannot write one |
| you can afford executable memory + cache flush discipline | you want to ship sooner |

A dynarec needs a VM block or kubridge **and** an I-cache flush after every generated
block (doc 07). An interpreter needs neither and is much easier to make correct.

## Step 3 — Renderer

| Guest GPU | Approach |
|-----------|----------|
| simple 2D/3D (DS) | software rasteriser + `vita2d` blit — no shaders, no `libshacccg` |
| PS1 GS / N64 RDP | HLE → vitaGL draws with per-state shaders |
| anything with a fixed state set | precompiled shader bank |

Software rasterisation is underrated: it sidesteps the whole shader pipeline and is
often *more* accurate.

## Step 4 — Memory

Budget explicitly and write the numbers down:

```
guest RAM + ROM        (the biggest, fixed)
recompiled code cache  (executable, several MB)
texture cache          (VRAM, tens of MB)
frame buffers          (2-3 x ~2 MB in CDRAM)
audio ring             (a few MB)
```

`ATTRIBUTE2=12` for the larger MAIN budget. If it does not fit, cut texture cache
resolution before cutting guest RAM.

## Step 5 — Frontend is part of the port

Ship these or it is not shippable:

- game/ROM selector with covers and metadata
- per-game settings (keyed by serial/ID) — titles differ wildly
- savestates + battery saves, both persisted
- full controller mapping, including analog deadzone
- a pause menu reachable with Start+Select
- on-screen messages for "BIOS missing", "ROM unreadable"

## Step 6 — Build hygiene

- record your **vitaGL build flags** in the README (the library is configuration-defined)
- bump a shader cache magic whenever you patch a shader
- keep `*.unstripped.elf` per build
- `-fno-optimize-sibling-calls` so stacks stay walkable

## Step 7 — Compatibility tracking from day one

A spreadsheet or an issues repo, one row per title, updated as you test. Emulator
projects live or die on this, and it stops you re-debugging the same game.

## Never

- never ship an emulator with no settings UI
- never assume "it boots" means "it plays"
- never let a patched shader reuse a cached compiled program
