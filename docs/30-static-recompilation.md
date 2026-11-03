# 30 — Static Recompilation

Lift an original closed-source binary to C ahead of time, then compile that C for
the Cortex-A9. No emulator, no source, and the game logic runs natively.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/static-recomp.md`, MIT). See [CREDITS](../CREDITS.md).

---

## 1. How it differs from the other routes

| Route | Source of truth |
|-------|-----------------|
| Decompilation port | recovered C source, matched to the original |
| Android `.so` load | the target platform's own ARM binary + a reimplemented OS |
| **Static recompilation** | **the original machine code, lifted to C** |
| Emulation | the original machine code, executed at runtime |

## 2. How the lift works (N64Recomp shape)

1. lift the original MIPS/x86 binary → C, with the recompiler tracking control flow
   statically
2. compile the generated C with the Vita toolchain (stock VitaSDK is fine; no
   softfp needed)
3. write the hardware layers from scratch:
   - **GPU**: original fixed-function rendering → GXM
   - **audio**: the console's audio microcode, run on the CPU
   - **input/display**: native pad, 960×544
4. verify the user's data file by hash at boot (the ecosystem norm)

The generated C is not "clean" — it is a transcription with explicit registers, jump
tables and memory accesses. That is fine; it compiles and it runs.

## 3. Reference projects (as published)

| Project | Source | Status |
|---------|--------|--------|
| Conker's Bad Fur Day | N64 | playable v1.0.0, 30 fps, user ROM hash-checked |
| The Binding of Isaac: Repentance | x86 PC | playable alpha, 960×544 |
| Gran Turismo 2 | PS1/MIPS | WIP — overlay removal, save backend swap |
| TimeSplitters: Future Perfect | PS2 | WIP — VIF/VU1 interpreter, GS-space preview |
| Monster Hunter Tri | Wii/PPC | shipping engine port, 60 fps target |
| vitarecomp / ps3recomp / psprecomp | Vita/PS3/PSP | toolkit stage |

Numbers worth knowing from the Wii work: ~1,650 hot functions moved to native C,
CDRAM ≈ 3 × 3.5 MiB frame slots + ~16 MiB converted models + ~47 MiB textures,
clang-21 + lld with `--pic-veneer` for the large code base, ThinLTO + PGO for
release, kubridge v0.3.1_**hotfix** plus a 4th-core unlocker required.

## 4. The order that works

```
recompile + boot on a desktop host
  → replace the GPU family by family with native producers
  → convert assets at load
  → add the Vita platform layer + precompiled shaders
  → profile, port hot functions, move work off the game thread
  → cut GPU time
  → ThinLTO + PGO release
```

Host-first is the rule. Get the main loop running on a PC before touching the
handheld; otherwise every bug is ambiguous between the lift and the platform.

## 5. Processor quirks you must reproduce

| Source CPU | Quirk |
|-----------|-------|
| R5900 (PS2) | FPU is not IEEE: clamp operands, round toward zero, no fused multiply-add |
| MIPS (PS1) | register-bound overlay loading must be replaced, not emulated |
| PPC (Wii/GC) | GX command stream; either intercept FIFO packets or replace at the call surface |
| x86 | flags, alignment tolerance, and `memcpy`-style unaligned access |

Do not "fix" these. A recomp that produces mathematically correct but behaviourally
different floats will desync physics and cutscenes.

## 6. Renderer strategy: two proven shapes

| Shape | Description | When |
|-------|-------------|------|
| Runtime codegen | state-keyed shader generation from GPU state (aurora-style), FIFO command processor | shader state space is unbounded or unknown |
| **Precompiled bank** | N precompiled `.gxp` shaders + an "uber shader", selected by a state key at runtime | fixed game, known state set |

The precompiled bank is faster and safer on Vita — no runtime compiler in the hot
path. ACGC (Animal Crossing GC → Vita) ships 52 precompiled Cg shaders + 1 uber
shader, embedded as a generated header, with 161 plain-C `GX*` functions and no FIFO
processor at all.

Runtime codegen earns its complexity only when you cannot enumerate the states.

## 7. Sizing and feasibility

Check before committing:

| Question | Why |
|----------|-----|
| does a lift tool exist for this console/ISA? | otherwise you are writing the recompiler first |
| how many distinct functions? | ~1,600 hot functions was tractable; 24 MB of code is not |
| how much VRAM for converted assets? | models + textures often exceed CDRAM |
| what cores do you need? | audio + render + game thread wants 3, sometimes a 4th |

## 8. When to use it

- closed-source game with no engine source
- a recomp or lift already exists (search "`<game>` recomp" / "`<game>` decomp")
- the original runs on a console with an established lift toolchain
- content is small enough after conversion

When not to: any target whose code volume dwarfs the device (a 24 MB PPC64
executable is a research project, not a port), or any game whose renderer cannot be
reduced to a finite shader bank.
