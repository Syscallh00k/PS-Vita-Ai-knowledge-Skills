---
name: static-recomp
description: Assess and run a static-recompilation port — check for an existing lift, get the loop running on a host first, then replace hardware family by family.
when: A closed-source game has no source, no ARM binary, but a lift/recomp tool or tree exists.
---

# Skill: Static Recompilation

## Step 1 — Confirm the route is even open

| Question | If "no" |
|----------|---------|
| is there a recompiler for that ISA? (N64, PS1/MIPS, PS2, x86, PPC) | you are writing the recompiler first — re-triage |
| does a decomp or recomp tree already exist? | search `"<game> recomp"`, `"<game> decomp"` |
| how much code is there? | ~1,600 hot functions is tractable; tens of MB is not |
| can the renderer be reduced to a finite shader set? | otherwise you need runtime codegen |

## Step 2 — Host first

Get the lifted code booting and running its main loop on a **desktop** build before
touching the Vita. Otherwise every bug is ambiguous between the lift and the platform.

**Gate:** the game loop advances and renders *something* on the host.

## Step 3 — Replace hardware family by family

Order that works:

```
GPU  → audio → input/display → assets → timing
```

- GPU: intercept at the call surface (`GX*`, `GTE`, draw submission) and emit your
  own draws; or intercept the command stream (FIFO/FIFO-packets) if the game mixes
  state too finely
- audio: run the console's audio microcode on the CPU
- input/display: native pad, 960×544

## Step 4 — Choose the shader strategy

| Strategy | When |
|----------|------|
| Precompiled bank (`psp2cgc` → `.gxp`, keyed at runtime) | state set is enumerable — **preferred on Vita** |
| Runtime codegen from GPU state | state set unbounded or unknown |

A precompiled bank keeps the runtime compiler out of the hot path. Ship it as a
generated header or an archive, not thousands of files.

## Step 5 — Reproduce the original CPU's float behaviour

Do not "fix" the emulated FPU. R5900 rounds toward zero and has no FMA; clamping and
rounding to match is what keeps physics and cutscenes in sync. Getting this
mathematically "more correct" desyncs the game.

## Step 6 — Convert assets at load, not at build

Console data is often already in a usable shape; converting at load keeps the user's
original files as the source of truth and avoids shipping derived data.

## Step 7 — Size the memory honestly

| Consumer | Typical |
|----------|---------|
| frame slots | 3 × ~3.5 MiB in CDRAM |
| converted models | ~16 MiB |
| textures | ~47 MiB |
| code + misc | the rest |

If the sum does not fit, cut texture resolution first (doc 24).

## Step 8 — Profile, then port hot functions

Only after it runs: move the hot 5% of lifted functions to hand-written native code,
move work off the game thread, then attack GPU time. ThinLTO + PGO are release-stage
tools, and LTO is risky — A/B on hardware.

## Step 9 — The user contract

- user supplies their own data file
- its hash is checked at boot, with an on-screen message on mismatch
- required plugins named with minimum versions (e.g. kubridge v0.3.1_hotfix)
- supported firmware stated if you use `MemProtect`-class calls (doc 37)

## Never

- never start on the Vita
- never replace the renderer at a higher level than the lowest stable software
  boundary — engine render logic should still be the game's
- never ship derived game data
