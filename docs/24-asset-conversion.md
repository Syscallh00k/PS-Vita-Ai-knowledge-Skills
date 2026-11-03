# 24 — Asset Conversion

Convert on the PC. Ship data that needs one `memcpy` on device.

---

## 1. Texture targets (SGX543)

| Use | Format | Note |
|-----|--------|------|
| colour, full quality | `U8U8U8U8_ABGR` | 4 B/texel, swizzle it |
| colour, 1-bit alpha | `U8U8U8U8` + shader alpha test | no blend cost |
| albedo without alpha | `U5G6B5`, `U4U4U4U4` | halves bandwidth |
| normal maps | `U8U8U8X8`, or 2-channel + reconstruct Z | 25–50% saving |
| lightmaps / LUTs | `U8U8U8U8`, few or no mips | 2D |
| PVRTC | `PVRTC2/4` | smallest, visibly lossy — background sets only |

Authoritative format list: `Sdk/include/psp2/gxm.h` (`SceGxmTextureFormat`).
Never guess a format from its name — channel order is not what you expect.

## 2. Swizzle

SGX reads Morton-order textures faster because neighbouring texels share cache
lines. Conversion is one pass at load (or entirely offline).

```c
static inline uint32_t MortonSpread(uint32_t v) {
    v &= 0x0000ffffu;
    v = (v | (v << 8)) & 0x00ff00ffu;
    v = (v | (v << 4)) & 0x0f0f0f0fu;
    v = (v | (v << 2)) & 0x33333333u;
    v = (v | (v << 1)) & 0x55555555u;
    return v;
}

/* y -> even address bits, x -> odd bits. Swapping them transposes every texture. */
[[nodiscard]] auto SwizzleIndex(uint32_t x, uint32_t y,
                               uint32_t w, uint32_t h) noexcept -> uint32_t;
```

Constraints: power-of-two dimensions, 8…4096. Non-square is a row of
`min(w,h)` Morton squares along the longer axis (see
`melee-Vita-vita-port/platforms/vita/game/texture_swizzle.h`).

## 3. Mipmaps

- generate offline; runtime generation costs frames and memory
- drop mips past the point where the texture is smaller than 8×8
- for UI and lightmaps, ship one level only

## 4. Meshes

- interleaved vertices, 4-byte aligned, ≤ 32 B/vertex
- typical: `position f16×4, normal i8×4, uv f16×2, tangent i8×4`
- indices: `U16` under 65536 verts, `U32` otherwise; triangles as **lists**
- `_Static_assert` every layout size and offset so a struct edit cannot silently
  corrupt a draw (Melee does this throughout `gx_render.c`)

## 5. Audio

| Source | Plan |
|--------|------|
| ADPCM / Vorbis / FSB banks | decode, then re-encode to ADPCM or ATRAC9 for streaming music |
| sample rate | resample once, offline; 32–44.1 kHz is enough |
| channels | downmix to stereo; mono for 3D-positioned sources |

Never resample at runtime per voice — that is a permanent tax for a one-time saving.

## 6. Video

Transcode to H.264 in MP4 (see doc 11): 960×544, high profile, 1.5–2.5 Mbps video,
AAC 96–128 kbps stereo, `+faststart`. This is usually the largest single size win
in a port.

## 7. One archive, not thousands of files

Thousands of small files destroy load times on Vita storage — PCSX AwO moved cover
art into a PSARC for exactly this reason.

```
 [header]  magic, version, entry count, index offset
 [index]   { u32 name_hash, u32 name_offset, u64 offset, u64 size, u32 flags }
 [names]   null-terminated, deduped
 [data]    16-byte aligned, optional per-entry compression
```

Requirements: O(1) lookup, sequential-read friendly, no per-asset `open()` during
play, ranges streamable with `SCE_O_NOWAIT`.

## 8. Build pipeline shape

```
 source data (read-only)
    ├─▶ extract       container → per-object blobs
    ├─▶ convert       texture / mesh / audio / video transforms
    ├─▶ pack          single indexed archive
    └─▶ verify        sizes, hashes, spot-render
```

Cache each stage by input hash so a 3 GB rebuild only touches what changed. This is
the difference between iterating and giving up.

## 9. Size budget worksheet

| Step | Typical saving |
|------|----------------|
| video transcode | 70–80% |
| texture half-res + 16-bit | 50–75% |
| audio resample/downmix | 30–40% |
| mesh strip + quantise | 40–60% |
| single archive | few % plus large load-time win |

Start from raw total, apply in that order, and check against your card budget before
committing to content scope.

## 10. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/texture_decoder.c` (GX formats),
  `game/texture_swizzle.h`, `game/gx_render.c`
- `Other Emulators/play-master/Source/gs/GSH_GXM/GSH_GXM_Texture.cpp`
- `Other Emulators/PCSX_AwO-master/tools/build_covers_psarc.py`, `fios_covers.cpp`
- `Sdk/include/psp2/gxm.h`
