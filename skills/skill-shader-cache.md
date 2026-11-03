---
name: shader-cache
description: Build a persistent shader permutation cache so runtime compilation happens in a training run, not in front of the player.
when: Anything compiles shaders at runtime, or a level stutters on first visit.
---

# Skill: Shader Cache

## Step 1 — Decide what belongs in the key

Hash only what changes compiled code:

```
key = hash(
    material class,
    alpha mode (opaque / masked / translucent),
    vertex format flags (skinned, tangents, lightmap uv),
    lighting mode,
    fog on/off,
    uv set count,
    any engine flag the shader reads)
```

Stage/state that is dynamic (depth func, blend when supported dynamically, uniforms)
does **not** belong in the key.

## Step 2 — Three tiers

```
RAM hash table  →  writable file (ux0:/data/<game>/shadercache/warm<N>.bin)
                →  packaged file (app0:/shadercache/warm<N>.bin)
                →  compile (last resort)
```

Writable beats packaged on duplicates. That lets a tester train locally without
rebuilding.

## Step 3 — Record format

```c
typedef struct GxrWarmHeader {
    uint32_t magic;
    uint32_t size;      /* program bytes, 4-byte padded */
    uint64_t hash;
    uint8_t  kind;      /* 0 = fragment, 1 = vertex */
    uint8_t  pad[3];
} GxrWarmHeader;
```

Rules that make it safe:

- **version the filename** (`warm5.bin`) so a format or precision change cannot
  load stale data
- bounds-check every record; report bad magic/kind/size with a byte offset
- validate with `sceGxmProgramCheck` on load
- deterministic precedence (writable → packaged, first duplicate wins)
- cap the file (Melee uses 16 MB) and log rejection reasons

## Step 4 — Train on device, ship the result

```
1. build with compilation enabled
2. play the level(s) with the cache in writable mode
3. pull ux0:data/<game>/shadercache/warm<N>.bin over FTP
4. validate every record, dedupe against the packaged cache, merge
5. ship the merged cache inside the VPK
```

Melee has a build switch for exactly this (`-WarmCacheVita <ip>`).

## Step 5 — Seal it in release

After warmup, disable compilation. A genuine miss then falls back to a known shader
instead of hitching. Count blocked misses and log them — that number should be zero
in a finished build.

## Step 6 — Reduce the problem before caching it

1. merge material classes (fewer is better than cached)
2. drop permutations nobody uses (masked variants of objects that are never masked)
3. compile a level's programs during its loading screen

## Diagnosis

| Symptom | Meaning |
|---------|---------|
| stutter every new area | cache misses; train more |
| stutter at the same spot every run | that shader never compiles successfully |
| cache file grows every session | keys are unstable (hashing a pointer or time) |
| cache loads but always misses | endianness or padding difference between writer and reader |

## Never

- never hash a pointer, an address, or anything time-dependent
- never ship a cache you did not validate record by record
- never let a cache format change silently reuse the old file
