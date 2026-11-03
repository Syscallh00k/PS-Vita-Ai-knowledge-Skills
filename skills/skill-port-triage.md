---
name: port-triage
description: Decide how a game can reach the Vita before writing code — binary ISA, engine, content size, and which of the five port families applies.
when: Someone says "port game X", or you are about to invest weeks in the wrong approach.
---

# Skill: Port Triage

## Step 1 — Identify the binary

```powershell
# what is it?
$b = [IO.File]::ReadAllBytes($path)[0..15]; ($b | % { $_.ToString('X2') }) -join ' '
```

| Clue | Meaning |
|------|---------|
| `7F 45 4C 46` + `e_machine` 40 | ARM ELF → binary-loading route open |
| `7F 45 4C 46` + 21 | PPC64 (PS3) → only reimplementation or recompilation |
| `53 43 45 00` | SCE SELF — Vita or PS3, check the inner ELF |
| PE, x86 | no direct route |
| packed/high entropy | the game packs itself; find the unpacker first |

Record: ISA, ELF class, endianness, entry point, segment sizes.

## Step 2 — Identify the engine

| Engine | Route |
|--------|-------|
| Godot 3 | Vita runtime exists → data patch |
| GameMaker | Vita runner ports exist → data patch |
| Unity (IL2CPP) | community runtime is fragile; usually not viable |
| Unreal Engine 3 | no Vita runtime → framework reimplementation |
| Unreal Engine 4+ | no |
| id Tech / Source / other with source | decomp/source port |
| custom in-house | depends entirely on a decompilation existing |

## Step 3 — Six questions

1. **ISA?** → which families are even possible
2. **Decompilation or source available?** → source port
3. **Engine has a shipping Vita port?** → data patch first, always
4. **Code vs. content by bytes?** content-heavy tolerates reimplementation
5. **Does it fit?** raw size × ~0.5 ≤ 3.5 GB, working set ≤ 250 MB
6. **How many distinct shader/material states?** tens is fine, thousands needs
   classification

## Step 4 — Pick the lowest-cost family that the answers allow

```
data patch  ─▶  source/decomp  ─▶  binary load  ─▶  framework  ─▶  emulation
 cheaper                                                          costlier
```

| Family | Precondition | Realistic ceiling |
|--------|--------------|-------------------|
| Data patch | engine has a Vita runtime | full game, minus unsupported effects |
| Source/decomp | compilable source | full game, high fidelity |
| Binary load | ARMv7 binary + reimplementable OS surface | full game |
| Framework | no source, finite content formats | vertical slice to substantial |
| Emulation | guest is much weaker than the host | whatever runs |

## Step 5 — Write the one-page verdict

Include:

- chosen family, and the rejected alternative with one sentence on why
- first three milestones, each with a date and a demo
- the stop condition — what result would make you abandon this, decided now

## Step 6 — Red flags that should stop you

- "we'll write an emulator" for anything newer than PS1
- "we'll decompile 24 MB of machine code" as the first step
- content that cannot fit even after aggressive recompression
- a shader system with thousands of permutations and no classification plan
- a plan whose first milestone is "the game runs"

## Never

- never commit to a family before measuring content size
- never start with gameplay; always start with the content/data path
- never assume the cheapest family applies without checking its precondition
