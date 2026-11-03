# 22 — Porting by Data Patching

When the engine already has a Vita port, you only ship content. Hours, not years.

Reference: `Other Ports/The-Dark-Queen-Of-Mortholme-psvita-main` — a Godot 3 game,
PCK repacked, wrapped in a VPK around a prebuilt Godot-Vita `eboot.bin`.

---

## 1. The premise

| Requirement | Why |
|-------------|-----|
| Vita port of that exact engine version | the runtime binary defines what content can load |
| Content format the engine accepts | you are not writing loaders |
| Willingness to ship someone else's runtime | you inherit its bugs and licence terms |

You gain: near-1:1 fidelity for a few hours of work. You lose: the ability to add
engine features, and shader coverage is whatever the port supports.

## 2. Godot PCK format

```
+0x00  char[4]  "GDPC"
+0x04  u32      pack format version (1..4)
+0x08  u32      major
+0x0C  u32      minor
+0x10  u32      patch
...    version >= 2: u64 file base offset, 16 reserved bytes
       u32 count, then directory entries:
         u32 path length, path bytes,
         u64 offset, u64 size, md5[16], u32 flags
```

Two details the reference tool gets right:

- **Embedded PCK.** A Godot `.exe` can carry the PCK appended; the authoritative
  locator is the 12-byte trailer `<u64 size><"GDPC">` at EOF, so
  `start = file_size - 12 - size`. Magic scanning is only the fallback, and it must
  validate version fields before believing a hit.
- **Path safety.** Reject `..`, absolute paths, and anything whose realpath escapes
  the output root. Extractors run on untrusted input by definition.

## 3. The three operations

| Operation | Steps |
|-----------|-------|
| create patch | index both PCKs; extract files that are new or changed into a folder |
| apply patch | extract original → overlay patch folder → repack using the original header |
| build VPK | copy `vpk_template/` (eboot.bin + `sce_sys/`) plus the repacked data |

`pck_patch_gui.py` in the reference does all three with a Tk UI and roughly 21 KB of
Python.

## 4. What "ported" actually meant for that game

- shaders partly disabled — the Vita renderer cannot compile the full set
- reflections kept on — cheap and high-impact in that renderer
- textures re-imported with Vita-appropriate compression and size
- a `vita_shader_startup_probe.gdc` scene that runs before real content to verify
  the pipeline

That last item is the single best idea in the whole repository: **ship a startup
probe that fails loudly.**

## 5. Adapting this pattern to a non-Godot engine

The same shape works whenever a Vita runtime exists:

```
 original container  ──▶  extract  ──▶  transform assets  ──▶  repack ──▶ VPK
```

Generalise the PCK script into:

1. a container reader (header + index + payload)
2. a per-asset transform table (texture recompress, audio resample, video transcode)
3. a writer that preserves the original header fields
4. a VPK wrapper with `param.sfo`, icon, LiveArea

## 6. Limits you must accept

| Limit | Consequence |
|-------|-------------|
| No new engine features | gameplay changes are content-only |
| Shader set is fixed | effects that need unsupported shaders get disabled |
| Runtime is a binary | you cannot fix its bugs, only avoid triggering them |
| Content still has to fit | storage budget unchanged from any other route |

## 7. Checklist before choosing this route

- [ ] Does a Vita port of this engine version exist and run?
- [ ] Does the game use features that port supports (check its changelog)?
- [ ] Can the assets be re-imported offline?
- [ ] Does the content fit the storage budget after re-import?
- [ ] Is the runtime's licence compatible with your distribution?

If any answer is "no", this route is a dead end and you should be looking at
docs 19/20/26 instead.

## 8. Sources

- `Other Ports/The-Dark-Queen-Of-Mortholme-psvita-main/pck_patch_gui.py`
- `.../README.md` (changelog: "some shaders have been disabled", "reflections enabled")
- `.../vpk_template/` (param.sfo, livearea, manifest)
