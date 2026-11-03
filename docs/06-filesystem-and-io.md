# 06 — Filesystem & I/O

Vita has no "current directory". Every path is absolute and device-prefixed.

---

## 1. Devices

| Device | Contents | Writable |
|--------|----------|----------|
| `app0:` | your VPK contents | **no** |
| `ux0:` | memory card / SD2Vita | yes |
| `ur0:` | internal system storage | yes (careful) |
| `uma0:` | USB / secondary mount | yes |
| `imc0:` | internal "imc" partition on some models | yes |
| `vs0:` | system partition | read-only for apps |
| `ux0:data/<game>/` | the conventional data directory | yes |
| `ux0:app/<TITLEID>/` | installed bubble | yes, via FTP |

Hard rule: **never open `app0:` for write**, and never assume the user's data
directory exists — create it.

## 2. Sync I/O

```c
SceUID fd = sceIoOpen("ux0:data/game/config.bin", SCE_O_RDONLY, 0);
if (fd < 0) { /* handle — do not continue with garbage */ }
SceOff size = sceIoLseek(fd, 0, SCE_SEEK_END);
sceIoLseek(fd, 0, SCE_SEEK_SET);
sceIoRead(fd, buffer, (SceSize) size);
sceIoClose(fd);
```

- `sceIoOpen` returns a negative `SceUID` on failure — check it
- `SCE_O_NOWAIT` gives non-blocking semantics for threaded streaming
- `sceIoMkdir`, `sceIoDopen`/`sceIoDread` for directories
- free-space queries use `sceAppMgrGetDevInfo` / `sceIoDevctl`

## 3. Async I/O: FIOS

`SceFios2` is the modern async layer used by every serious port:

| Feature | Benefit |
|---------|---------|
| async reads with completion callbacks | no main-thread stall |
| `sceFiosArchive` mount | one call, whole archive visible as a directory |
| `sceFiosFHOpenSync` / `sceFiosFHReadSync` | simple API if you don't need async |
| FIOS overlays (`SceFiosOverlay`) | **remap paths at runtime** — this is how Android ports redirect `/data/...` to `ux0:data/game/...` |

`soloader-template` ships a small `fios` overlay library (`lib/fios/fios.c`) that
does exactly one thing: make a foreign game's hardcoded paths resolve inside your
data directory.

## 4. Streaming architecture that avoids hitches

```
 main thread ──request──▶ [bounded queue, cap ~32] ──▶ IO thread (SCE_O_NOWAIT)
      ▲                                                      │
      └──────────── completion poll / callback ◀─────────────┘
```

Rules learned from the reference ports:

- one IO thread, not many — card seeks are the bottleneck
- bound the queue; dropping a request and drawing a lower LOD beats stalling
- dispatch completions **on the game thread** (Melee's
  `melee_vita_platform_poll()`), so callbacks run where the game expects
- never open thousands of small files during play — see doc 24 for the archive
  design

## 5. Path conventions for a port

```c
#define GAME_DATA_DIR  "ux0:data/<game>/"
#define GAME_APP_DIR   "app0:/"
#define GAME_SAVE_DIR  "ux0:data/<game>/saves/"
```

Keep them in one header. Android loaders keep them in CMake defines
(`-DDATA_PATH=...  -DSO_PATH=...`) so the same source builds for multiple ports.

## 6. Save data

Two options:

1. **Plain files** in `ux0:data/<game>/saves/` — simple, visible, easy to back up.
2. **`SceAppUtil` save data** — proper save slots, shows in the content manager,
   and required if you want the system to treat saves as saves. `sceAppUtilSaveData`
   with a slot list; this is what retail-style titles use.

Never write saves into `app0:` and never assume `ux0:data` exists on first run.

## 7. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/game/dvd.c` — async streaming thread
- `soloader-template/lib/fios/fios.c` and `source/reimpl/io.c`
- `Sdk/include/psp2/io/fcntl.h`, `io/stat.h`, `io/dirent.h`, `fios2kernel.h`
