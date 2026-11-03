# 27 — Provenance & Distribution

How to publish a port that does not redistribute what it should not.

---

## 1. The rule that keeps projects alive

> Ship **code and tools**. Never ship game data, keys, or proprietary modules.

Every reference project in this workspace follows it:

| Project | What it ships | What the user supplies |
|---------|---------------|------------------------|
| Melee Vita | engine code + VPK | their own `GALE01.iso` |
| Raider | loader + `datafiles.zip` (its own assets) | `.apk` with `libmain.so` + assets |
| PCSX AwO | emulator | their own game images, optional BIOS |
| Dark Queen patch | patch files + VPK template | the Steam/itch.io game |
| soloader-template | boilerplate loader | the game's `.so` + data |

Notice the pattern: the user always brings the copyrighted part, and the repository
brings the machinery that consumes it.

## 2. What "converted content" means legally

Two distinct actions:

1. **Transform on the user's machine.** A tool reads their copy and writes derived
   data. The derived data never enters your repository and is never distributed.
   This is the standard, defensible shape used by every patcher and asset converter.
2. **Transform and publish.** You redistribute derived assets. This is what gets
   projects taken down, regardless of how much was transformed.

Design for (1). If a build needs pre-converted data, ship the *converter* and a
manifest of hashes, not the data.

## 3. Things that must never appear in a repository

- game assets of any kind (textures, audio, video, packages)
- decryption keys, key files, `keys.txt`, licence blobs
- retail `eboot.bin` / `SELF` files you did not build
- firmware blobs, `PS3UPDAT.PUP`, Vita system modules
- `libshacccg.suprx` (users extract it themselves from their own console)
- BIOS images, ROMs, ISOs
- someone else's runtime binary used as your `eboot.bin`

That last one is subtle: wrapping a third-party engine runtime is common and legal
only if that runtime's licence allows redistribution. Check, and credit.

## 4. Credit the chain

Every port stands on:

- the SDK and its community (`VitaSDK`, `vitaGL`, `vita2d`, `vitaShaRK`,
  `SceShaccCgExt`, `kubridge`, `taiHEN`)
- the reverse engineers who mapped the platform
- the decompilation project, if there is one
- the original game's developers and publisher (name them; do not pretend otherwise)

A credits table in the README is not politeness, it is how the ecosystem stays
cooperative.

## 5. User-facing honesty

Two lines that belong in every port's README:

- "You need your own copy of X. This project does not include any game data."
- "This is experimental; back up your saves."

Also state real compatibility: which game version/region, which firmware, which
plugins are required (`kubridge`, `libshacccg`, overclock), and what is known broken.

## 6. Licences

| Component | Typical licence | Obligation |
|-----------|-----------------|------------|
| your port code | your choice (GPL/MIT) | state it |
| decompiled game code | usually **not** yours to relicense | keep it separate, say so |
| third-party libraries | MIT/GPL/zlib | keep their notices |
| engine runtime you wrap | check specifically | may forbid redistribution |

Melee's project handles this correctly and explicitly: the port code and the
decompiled game code carry different licences, and the README says so.

## 7. Release checklist

- [ ] no game data in the repo or the release archive
- [ ] no keys, no firmware, no retail binaries
- [ ] third-party notices present
- [ ] README states requirements and known issues
- [ ] version tag matches the VPK metadata
- [ ] the build reproduces from a clean checkout

## 8. Sources

- `Other Ports/melee-Vita-vita-port/README.md` (requirements, scope, licensing note)
- `Other Ports/raider-vita-master/README.md` (delisting disclaimer, "do not ask how
  to acquire game data")
- `Other Ports/The-Dark-Queen-Of-Mortholme-psvita-main/README.md` (patch-not-game
  distribution model)
