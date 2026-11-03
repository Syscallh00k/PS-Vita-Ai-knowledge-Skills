---
name: data-patch-port
description: Port a game whose engine already runs on Vita by patching and repacking its content — container parsing, asset transform, VPK wrapping.
when: Godot/GameMaker/other Vita-hosted engine game, or any engine with a working Vita runtime.
---

# Skill: Data-Patch Port

## Step 0 — Precondition, stated plainly

There must be a working Vita runtime for **that engine version**. If there is not,
this skill does not apply and you are looking at doc 19, 20 or 26 instead.

## Step 1 — Identify the container

| Engine | Container | Clue |
|--------|-----------|------|
| Godot 3 | `.pck` (or appended to the `.exe`) | magic `GDPC` |
| GameMaker | `data.win` | magic `FORM` |
| Unity | `resources.assets` / bundles | UnityFS header |
| custom | whatever the exe reads | start at the file it opens first |

For Godot, the authoritative locator for an embedded PCK is the 12-byte trailer:

```
tail = <u64 size><"GDPC">    →  pck_start = file_size - 12 - size
```

Validate before trusting: `fmt ∈ 1..4`, plausible major/minor/patch.

## Step 2 — Extract with path safety

Reject empty segments, `.`, `..`, drive letters, and any path whose realpath escapes
the output directory. Extractors run on untrusted input by definition.

## Step 3 — Transform assets

| Asset | Transform |
|-------|-----------|
| textures | re-import at Vita-appropriate size/compression |
| audio | resample (32–44.1 kHz), downmix to stereo |
| video | transcode to H.264/MP4 (doc 11) |
| shaders | prefer the engine's built-in set; disable unsupported features |
| fonts/UI | author at 960×544, integer scaling |

## Step 4 — Repack preserving the original header

Write the index and payload back using the **original** header fields (format
version, base offset, alignment). Engines validate these; a "cleaner" header that
differs will be rejected.

## Step 5 — Add a startup probe

Ship a tiny scene that runs before real content and exercises the renderer: one
textured, lit, blended quad plus whatever shader features you rely on. If it fails,
the log says so instead of the player seeing a black screen three scenes later.

## Step 6 — Wrap in a VPK

```cmake
vita_create_vpk(game.vpk ${VITA_TITLEID} eboot.bin
    VERSION ${VITA_VERSION} NAME ${VITA_APP_NAME}
    FILE sce_sys/icon0.png sce_sys/icon0.png
    FILE ${CMAKE_BINARY_DIR}/game.pck game.pck)
```

The runtime `eboot.bin` comes from the engine's Vita port — check its licence
permits redistribution before shipping it inside your package.

## Step 7 — Verify on hardware

- boots to the startup probe (proves the runtime + your repack)
- content loads (proves the index rewrite)
- audio and video play (proves the transforms)
- storage fits (sum the archive before you promise anything)

## Diagnosis

| Symptom | Cause |
|---------|-------|
| runtime starts, game does not | index/header rewrite invalid |
| textures missing | re-import produced a format the runtime rejects |
| wrong colours | texture compression or channel order |
| black screen on specific scenes | unsupported shader feature |
| audio silent | sample rate or channel count mismatch |
| installs, immediate exit | runtime expects data at a specific path |

## Never

- never modify the engine runtime binary
- never redistribute the original game data
- never skip the startup probe — it is the cheapest bug-catching device in this
  whole workflow
