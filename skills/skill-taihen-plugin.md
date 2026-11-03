---
name: taihen-plugin
description: Write a suprx/skprx plugin that hooks or patches a running app — skeleton, exports.yml, safety rules, and how to not brick a console.
when: You need to observe or modify a running game, including one you cannot rebuild.
---

# Skill: taiHEN Plugin

## Step 1 — Pick the module type

| Type | Runs in | API prefix | Use |
|------|---------|-----------|-----|
| `.suprx` | userland | `sce*` | hooking app-level functions, overlays |
| `.skprx` | kernel | `ksce*` | memory patching, import hooks, drivers |

Kernel modules are more powerful and more dangerous. Start userland unless you need
physical memory or import tables.

## Step 2 — Skeleton

```c
#include <taihen.h>
#include <vitasdk.h>

static tai_hook_ref_t g_ref;
static int g_uid = -1;

static int Hooked(int a1, int a2) {
    LogWrite("call(0x%08X, 0x%08X)", a1, a2);
    int ret = TAI_CONTINUE(int, g_ref, a1, a2);
    LogWrite(" -> 0x%08X", ret);
    return ret;
}

void _start() __attribute__((weak, alias("module_start")));

int module_start(SceSize argc, const void* args) {
    tai_module_info_t info;
    info.size = sizeof(info);
    if (taiGetModuleInfo(TAI_MAIN_MODULE, &info) < 0) return SCE_KERNEL_START_SUCCESS;
    g_uid = taiHookFunctionOffset(&g_ref, info.modid, 0, 0x0, 1, Hooked);
    return SCE_KERNEL_START_SUCCESS;
}

int module_stop(SceSize argc, const void* args) {
    if (g_uid >= 0) taiHookRelease(g_uid, g_ref);
    return SCE_KERNEL_STOP_SUCCESS;
}
```

The `_start` weak alias is not optional — without it the module has no entry.

## Step 3 — exports.yml

```yaml
my_plugin:
  attributes: 0
  version:
    major: 1
    minor: 1
  main:
    start: module_start
    stop: module_stop
```

Needed for the build to produce a valid module. Exporting functions to other modules
adds library entries to this file.

## Step 4 — Safety rules

1. **Always release hooks** in `module_stop`.
2. **Never block** in `module_start` — load order will deadlock.
3. **Verify before writing** — check a byte signature at the target offset; a build
   difference means you would overwrite real code.
4. **Fail safe** — if anything looks wrong, log and do nothing. A half-applied patch
   is worse than no patch.
5. **Keep a recovery path** — you can always boot with the tai folder renamed or over
   USB/FTP.

## Step 5 — Study the local examples

| Tool | Technique |
|------|-----------|
| `Tools/base_hooker` | `taiHookFunctionOffset`, `TAI_CONTINUE`, hex-dumping pointer args |
| `Tools/noASLR` | `taiHookFunctionImportForKernel`, patching instruction bytes (`movs r0,#0`, NOPs) |
| `Tools/ioPlus` | exporting a service others call, with `exports.yml` |
| `Tools/physmem-dumper` | kernel physical-memory walk |
| `Tools/PSVita-RTC-set` | minimal-risk kernel write |
| `Tools/unity_debug` | hooking engine internals by offset + UTF-16 decoding |
| `Tools/FAGDec` | cooperating kernel + user plugin pair |

## Step 6 — Debug it

- `sceClibPrintf` / `ksceDebugPrintf` → kernel log, readable over
  `Tools/PrincessLog`
- log every hook UID; a negative UID means "patch never applied"
- test on a spare console, or with a config you can revert remotely

## Step 7 — When not to write a plugin

| Situation | Better answer |
|-----------|---------------|
| your own app needs a setting | add a menu |
| you need path redirection for a port | FIOS overlay |
| a bug in someone's source | patch the source |
| you want to observe a retail game | plugin (this is the case for it) |

## Never

- never ship a plugin with an unguarded write to a hardcoded offset
- never leave a hook installed on exit
- never assume the user's other plugins do not already hook the same function
