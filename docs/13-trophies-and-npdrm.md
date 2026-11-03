# 13 — Trophies & NPDRM

Optional polish that turns a homebrew into something that feels like a release.

---

## 1. Trophies

Two pieces: a **trophy pack** shipped in the VPK, and **runtime calls** from the
game.

```
sce_sys/trophy/<CONTENT_ID>_00/TROPHY.TRP
```

CMake (from the Raider port, which ships two packs for two games):

```cmake
vita_create_vpk(Raider.vpk ${VITA_TITLEID} eboot.bin
    FILE ${CMAKE_SOURCE_DIR}/sce_sys/trophy/RAID00001_00/TROPHY.TRP
         sce_sys/trophy/RAID00001_00/TROPHY.TRP
    FILE ${CMAKE_SOURCE_DIR}/sce_sys/trophy/RAID00002_00/TROPHY.TRP
         sce_sys/trophy/RAID00002_00/TROPHY.TRP)
```

Runtime:

```c
sceNpTrophyCreateContext(&ctx, 0, 0, 0);
sceNpTrophyRegisterContext(ctx, ...);
sceNpTrophyUnlockTrophy(ctx, handle, trophy_id, &platinum_id);
sceNpTrophyDestroyContext(ctx);
```

Rules that avoid wasted evenings:

- trophy IDs are 1-based within the pack and must match the pack exactly
- unlocking a trophy the pack does not define fails with a negative code — log it
- a port that emulates a save file can derive trophies from the same events, which
  is usually better than inventing new ones
- the pack must be generated before the VPK is built; it is not created at runtime

## 2. NPDRM (license checks)

`Sdk/include/psp2/npdrm.h` + `libSceNpDrm_stub.a` expose:

| Call | Purpose |
|------|---------|
| `sceNpDrmIsAvailable` | is there a valid license for this content id |
| `sceNpDrmGetRifName` / `sceNpDrmGetRifInfo` | license file details |
| `sceNpDrmGenerateRif` | (kernel-side, retail flows) |

For homebrew and ports you generally **do not** need NPDRM. It exists in the SDK,
and it is the mechanism retail content uses to bind to an account. Two honest notes:

- calling it without a real license returns errors you must handle
- do not build a port whose operation depends on a license check you cannot satisfy

`Tools/NoNpDrm-master` is a community plugin that fakes licenses for retail titles.
It is unrelated to homebrew development and irrelevant to anything you build — keep
it out of build/test workflows so results are not contaminated by a plugin.

## 3. Save data & the content manager

If you want saves to appear in the system content manager (and survive a
reinstall), use `SceAppUtilSaveData` slots rather than loose files. Trophy
unlocks typically ride along with save data, so decide both at once.

## 4. Sources

- `Other Ports/raider-vita-master/loader/trophies.c` and its `CMakeLists.txt`
- `Sdk/include/psp2/npdrm.h`
- `Tools/NoNpDrm-master/` (reference only — not part of any build)
