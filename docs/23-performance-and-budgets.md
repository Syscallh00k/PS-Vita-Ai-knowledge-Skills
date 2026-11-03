# 23 — Performance & Budgets

Numbers to design against, and how to measure whether you are hitting them.

---

## 1. Frame budget

| Target | Total | CPU-side | GPU-side |
|--------|-------|----------|----------|
| 60 fps | 16.6 ms | ~10 ms | ~16 ms overlapped |
| 30 fps | 33.3 ms | ~22 ms | ~33 ms overlapped |

Honest starting assumption for a 3D port: **30 fps at 480×272 or 576×328 internal
resolution**, upscaled to 960×544. Anything better is a bonus earned by spending
where it counts.

## 2. What actually costs the frame, in order

1. fragment shader complexity × overdraw (fill rate is the usual killer)
2. draw call and state-change count (blend changes force new fragment programs)
3. vertex processing of unbatched meshes
4. CPU scene traversal (transform, cull, visibility) — where engine ports die first
5. texture bandwidth (no mips, no swizzle, or both)

## 3. Memory budget

| Pool | Budget |
|------|--------|
| code + rodata | 20–30 MB |
| streamed geometry | 40–60 MB |
| resident textures | ≤ 80 MB |
| audio | 4–8 MB |
| I/O staging | 16–32 MB |
| heap remainder | whatever your `_newlib_heap_size_user` leaves |

Track peaks per subsystem in dev builds and print a table periodically. Everyone
grows this eventually; start with it.

## 4. Instrumentation that pays for itself

| Metric | Cost to add | Value |
|--------|-------------|-------|
| zone timers (named regions, ring buffer, overlay) | minutes | finds the 8 ms outlier |
| draw calls per frame | one counter | reveals batching regressions |
| shader cache hits per tier | one counter each | explains stutter |
| texture bytes resident | one counter | catches leaks |
| I/O bytes and ms per request | one wrapper | explains hitches |
| CPU/GPU overlap (sync on/off A/B) | one build flag | tells you who is the bottleneck |

## 5. Resolution scaling is the primary lever

Order of operations when a scene is slow:

1. drop internal resolution one tier (960×544 → 720×408 → 576×328 → 480×272)
2. disable expensive per-material effects
3. reduce shadow map size / cascade count
4. reduce draw distance and streamed LOD
5. only then rewrite shaders

Frames requiring exact texel alignment (render-target readback, copy-to-texture,
video upload) must render at native — never scale them and hope.

## 6. Shader stutter is a separate problem

A material-heavy engine will compile programs on first encounter. Mitigations in
order of value:

1. fewer, broader material classes
2. pre-train the cache on device, pull it back over FTP, ship it in the VPK
3. warm the programs a level needs during its loading screen
4. seal compilation in release so a miss degrades to a fallback instead of a hitch

## 7. Regression benchmarks to keep

| Scene | Measures |
|-------|----------|
| clear + 1 draw | baseline present cost |
| one full-screen textured quad at 4× overdraw | fill rate |
| 2000 instanced meshes, 1 material | vertex/throughput |
| 2000 meshes, 50 materials | state-change cost |
| level flythrough | streaming + traversal |
| menu | 2D path |

If any of these moves more than 10% between builds, stop and find out why.

## 8. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/README.md` (resolution scale
  options and the native-fallback reason flags)
- `.../platforms/vita/game/profiler_live.c`, `gxm_resolution.h`
- `Other Emulators/PCSX_AwO-master/frontend/vita/standalone.c` (present paths, LUTs)
