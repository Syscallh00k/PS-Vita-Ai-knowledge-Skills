---
name: build-vpk
description: Turn a Vita project into an installable bubble — CMake wiring, SELF signing, LiveArea assets, param.sfo, and the failure modes.
when: The project builds but will not install, or a new project needs packaging.
---

# Skill: Build a VPK

## Step 1 — The four variables

```cmake
set(VITA_APP_NAME "My Port")
set(VITA_TITLEID  "MYPT00001")     # 9 chars, uppercase alnum
set(VITA_VERSION  "00.10")         # two-part
set(VITA_MKSFOEX_FLAGS "-d ATTRIBUTE2=12")
```

Misspell one and you get a VPK that installs with a blank name or refuses to launch.

## Step 2 — SELF then VPK

```cmake
vita_create_self(eboot.bin app)
vita_create_vpk(app.vpk ${VITA_TITLEID} eboot.bin
    VERSION ${VITA_VERSION} NAME ${VITA_APP_NAME}
    FILE sce_sys/icon0.png sce_sys/icon0.png
    FILE sce_sys/livearea/contents/bg0.png sce_sys/livearea/contents/bg0.png
    FILE sce_sys/livearea/contents/startup.png sce_sys/livearea/contents/startup.png
    FILE sce_sys/livearea/contents/template.xml sce_sys/livearea/contents/template.xml)
```

`UNSAFE` on `vita_create_self` only if you patch code pages at runtime. VM-block JIT
does not need it.

## Step 3 — Asset sizes

| File | Size |
|------|------|
| `icon0.png` | 128×128 |
| `bg0.png` | 840×500 |
| `startup.png` | 280×158 |
| `pic0.png` | varies (used by some views) |

Wrong sizes produce stretched or blank art, not an error.

## Step 4 — Install and verify

```powershell
curl -T build/app.vpk ftp://$env:PSVITAIP:1337/ux0:/
echo "launch MYPT00001" | nc $env:PSVITAIP 1338
```

Or copy the VPK to the card and install with VitaShell.

## Step 5 — Failure table

| Symptom | Cause |
|---------|-------|
| install fails silently | bad title ID characters, or `param.sfo` not generated |
| icon blank | `icon0.png` missing from the VPK |
| launches, exits instantly | not a SELF, or `ATTRIBUTE2` missing |
| launches, black screen | app started but drew nothing — see skill `first-frame` |
| "corrupted" | VPK was re-zipped with the wrong layout; `sce_sys/` must be at root |
| title shows garbage | `VITA_APP_NAME` empty or non-ASCII without encoding |
| works over FTP, not as VPK | you are writing to `app0:`, which is read-only |

## Step 6 — Keep iteration cheap

For development, FTP `eboot.bin` directly into `ux0:/app/<TITLEID>/`. Uploading a
full VPK every time is minutes per cycle; the eboot is seconds.

## Never

- never ship a VPK whose `eboot.bin` you did not build
- never put save files or config inside `app0:`
- never exceed ~500 MB in a single VPK — split content into a separate data archive
  and read it from `ux0:`
