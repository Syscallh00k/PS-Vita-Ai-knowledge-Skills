# 18 — taiHEN Plugin Development

Plugins patch a *running* system — yours or someone else's. This is the most
powerful and least forgiving area of Vita development.

---

## 1. Module types

| Extension | Runs in | API prefix | Example |
|-----------|---------|-----------|---------|
| `.suprx` | userland | `sce*` | graphics/settings overlays |
| `.skprx` | kernel | `ksce*` | memory patching, hooks, drivers |

Both are SELF modules loaded by taiHEN from `config.txt`:

```
*KERNEL
ux0:tai/kubridge.skprx
ux0:tai/ioPlus.skprx
*main
ux0:tai/my_patch.skprx
```

`*KERNEL` entries load at boot; `*main` loads into every app process.

## 2. Skeleton

```c
#include <taihen.h>
#include <vitasdk.h>

static tai_hook_ref_t g_hook_ref;
static int g_hook_uid = -1;

static int HookedFunc(int a1, int a2) {
    LogWrite("HookedFunc(0x%08X, 0x%08X)", a1, a2);
    int ret = TAI_CONTINUE(int, g_hook_ref, a1, a2);
    LogWrite(" -> 0x%08X", ret);
    return ret;
}

void _start() __attribute__((weak, alias("module_start")));

int module_start(SceSize argc, const void* args) {
    tai_module_info_t info;
    info.size = sizeof(info);
    taiGetModuleInfo(TAI_MAIN_MODULE, &info);
    g_hook_uid = taiHookFunctionOffset(&g_hook_ref, info.modid, 0, 0x0, 1, HookedFunc);
    return SCE_KERNEL_START_SUCCESS;
}

int module_stop(SceSize argc, const void* args) {
    if (g_hook_uid >= 0) taiHookRelease(g_hook_uid, g_hook_ref);
    return SCE_KERNEL_STOP_SUCCESS;
}
```

That is `base_hooker` almost verbatim, including the `_start` weak alias which is
required for the module entry to be found.

## 3. Rules that prevent bricking

1. **Always release hooks in `module_stop`.** A leaked hook corrupts the import
   table for every later launch.
2. **Never block in `module_start`.** Load order matters; waiting on another module
   deadlocks the boot.
3. **Validate before patching.** Check a byte signature before writing an offset —
   a single build difference means you are overwriting real code.
4. **Keep a panic path.** If your patch fails, do nothing and log. A plugin that
   half-applies is worse than one that does not load.

## 4. What the tools in this workspace demonstrate

| Tool | Technique worth stealing |
|------|--------------------------|
| `base_hooker` | `taiHookFunctionOffset` + `TAI_CONTINUE` + hex dump of pointer args |
| `noASLR` | `taiHookFunctionImportForKernel` and patching instruction bytes (`movs r0, #0`, `nop`s) |
| `ioPlus` | exports a service other code can call, plus `exports.yml` |
| `physmem-dumper` | kernel-side physical memory walking |
| `PSVita-RTC-set` | kernel-side RTC write with a minimal risk surface |
| `unity_debug` | hooking engine internals (IL2CPP) by offset and decoding UTF-16 strings |
| `FAGDec` | cooperating user + kernel plugin pair |

`ioPlus` ships two versions and a file literally named
`README_DO_NOT_USE_THIS_USE_0.1.txt` — a useful reminder that a newer plugin is not
automatically a safer one, and that shipping a note about it is kinder than
shipping silence.

## 5. When a plugin is the right answer

| Situation | Plugin? |
|-----------|---------|
| Your own homebrew needs a setting | no — just add a menu |
| You need to observe a retail game's behaviour | yes |
| You need to redirect file paths for a port | maybe — FIOS overlays are safer |
| You want to fix another developer's bug at runtime | no — patch the source |
| You are reverse-engineering an unknown function | yes, with logging hooks |

## 6. Debugging a plugin

- `sceClibPrintf` / `ksceDebugPrintf` goes to the kernel log, visible over
  `Tools/PrincessLog` or a debug UART
- a plugin that crashes at boot means no app starts — keep a recovery path (boot
  with the tai folder renamed or via a safe-mode plugin)
- test on a spare console or with a config you can revert over FTP/USB

## 7. Sources

- `Tools/base_hooker/src/base_hooker.c`, `base_hooker.yml`
- `Tools/noASLR/src/main.c`
- `Tools/ioPlus/ioPlus-0.1/src/ioplus.c` + `exports.yml`
- `Tools/unity_debug/src/unity_debug.c`
- `Tools/FAGDec/src/kplugin/`, `src/uplugin/`
