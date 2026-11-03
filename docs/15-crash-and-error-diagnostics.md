# 15 — Crash & Error Diagnostics

Vita failures are quiet. Learn the signatures.

---

## 1. Error code anatomy

Most SDK failures return a negative `int` of the form `0x8xxxxxxx`:

```
0x8 0 FFFFFF
 │ │   └── module/detail bits
 │ └────── class
 └──────── "error" marker
```

Examples you will actually see:

| Code | Meaning |
|------|---------|
| `0x80010002` | invalid argument / bad handle |
| `0x8001000D` | already exists |
| `0x8010113D` | (SceGxm) invalid/uninitialised usage |
| `0x80870002` | (SceNpTrophy) invalid context |
| `0x80101A04` | (SceFios) not found |
| `0x8002D0xx` | module load failures |

Resolve them properly instead of guessing:

- `Tools/PSVita-error-code-resolver` reads the firmware's `error_table.bin` and
  maps a code to a string (`error_code_to_string`)
- `Sdk/include/psp2/error.h` has the common base codes

## 2. Symptom → cause table

| Symptom | Most likely cause |
|---------|-------------------|
| returns to LiveArea immediately | bad SELF, `ATTRIBUTE2` missing, or panic before display init |
| app exits with no message | an `abort()`/assert triggered — override them (see §4) |
| hang at ~5 s | IO completion dispatched on the wrong thread |
| hang on first draw | shader compile blocking, or patcher memory exhausted |
| freeze when closing | teardown joined a thread that owns a callback |
| crash after minutes | GPU or heap exhaustion |
| crash only after resolution change | stale relaunch state not validated |
| works on Vita3K, dies on hardware | timing, memory ceiling, missing plugin |

## 3. Kernel dumps

- `Tools/Kdumper` dumps kernel memory (with a PC-side fetch script
  `RUN_TCP_DUMP.bat`)
- `Tools/kdump-extract` extracts a dump for analysis
- `Tools/physmem-dumper` dumps physical ranges
- `vita-parse-core` (referenced by `soloader-template`'s CMake `dump` target)
  symbolises a core file against an ELF

**Save the ELF for every build you put on the device.** A dump without symbols is a
hex blob.

## 4. Fail loudly, on purpose

Fresh projects crash silently because newlib's `abort` and assert handler just
stop. Take them over:

```c
void abort(void) {
    LogWrite(LOG_ERROR, "abort() called");
    sceKernelExitProcess(0);
    while (1) sceKernelDelayThread(1000000);
}

void __assert_func(const char* file, int line, const char* fn, const char* expr) {
    LogWrite(LOG_ERROR, "assert failed: %s at %s:%d (%s)", expr, file, line, fn);
    abort();
}
```

Play!'s `main.cpp` does exactly this. Five lines that turn "it just closes" into a
filename and a line number.

## 5. Boot-time validation

```c
#define CHECK(cond, msg) do { if (!(cond)) { \
    LogWrite(LOG_ERROR, "BOOT FAIL: " msg); \
    sceKernelExitProcess(0); } } while (0)

CHECK(module_loaded("kubridge"), "kubridge.skprx missing");
CHECK(file_exists("ux0:data/game/base.archive"), "data missing");
CHECK(file_exists("ur0:data/libshacccg.suprx"), "shader compiler missing");
```

Every port that ships well validates its prerequisites before touching the GPU.

## 6. Reproduce on PC first

Boot the same VPK in `Emulator/Vita3K.exe`. If it fails identically there, it is a
logic or packaging bug and you get a real debugger. If it only fails on hardware,
the search space collapses to timing, memory, GPU, and plugins.

## 7. Sources

- `Tools/PSVita-error-code-resolver/src/main.c`
- `Tools/Kdumper/`, `Tools/kdump-extract/`, `Tools/physmem-dumper/`
- `Other Emulators/play-master/Source/ui_vita/main.cpp` (abort/assert overrides)
- `soloader-template/source/utils/dialog.c` (`fatal_error` on-screen)
