# 17 — NID & Import System

Vita system libraries are addressed by **NID**, not by name. This is the mechanism
behind stubs, hooks, and every "why won't it link" error.

---

## 1. What a NID is

A 32-bit hash of a function's signature (name + prototype). The firmware's
`libSceGxm` exports `sceGxmBeginScene` under a NID like `0xA8E2C4B8`; the stub
library contains only that NID, and the loader resolves it at boot.

Consequences:

- you cannot call a function that is not in a stub you link against
- you cannot hook a function by name at the ABI level — only by NID or address
- the same API can have different NIDs across firmware versions (rare but real)

## 2. Modules, libraries, functions

```
module (SceGxm)  ──▶  library (SceGxm)  ──▶  function (NID 0xA8E2C4B8, sceGxmBeginScene)
```

`Sdk/include/psp2/...` headers declare the C prototypes; the `libSce*_stub.a`
archives supply the NIDs. `Sdk/include/vita-import.h` and
`vita-toolchain-public.h` describe the import format the linker uses.

## 3. YAML-driven builds

Modern VitaSDK builds modules from YAML descriptions:

```yaml
# exports.yml — a plugin exporting functions
base_hooker:
  attributes: 0
  version:
    major: 1
    minor: 1
  main:
    start: module_start
    stop: module_stop
```

The build tools (`vita-import`, part of the toolchain) turn these into
`*.yml`-driven import/export tables embedded into the module. Every plugin in
`Tools/` that has an `exports.yml` works this way: `base_hooker`, `noASLR`,
`physmem-dumper`, `PSVita-RTC-set`, `unity_debug`, `ioPlus`.

## 4. NID → name resolution

`Tools/THGN` ("That Hookers Got NIDs") does the useful trick: given an ELF and a
database of library YAMLs, it maps NIDs back to readable names.

Flow:

1. parse the ELF's import/export tables (`import/vita-import-parse.c`)
2. look each NID up in a bundled YAML DB (`findNameByNid`)
3. emit hook entries in the format a plugin/hooker consumes

Why it matters: a stripped module's imports are just numbers. THGN turns
`0x6D9F1C4E` into `sceKernelGetProcessId`, which is the difference between guessing
and knowing.

## 5. Hooking by NID vs. by offset

| Method | API | Stable across versions |
|--------|-----|------------------------|
| Export hook by NID | `taiHookFunctionImport` / `taiHookFunctionImportForKernel` | yes-ish |
| Export hook by offset | `taiHookFunctionOffset` | fragile — offset changes per build |
| Direct address patch | `kuKernelCpuUnrestrictedMemcpy` | most fragile, needs noASLR |

`base_hooker` uses `taiHookFunctionOffset` after finding the main module with
`taiGetModuleInfo(TAI_MAIN_MODULE, &info)`. That is the right tool for
"single known build" debugging, not for shipping to users.

## 6. Practical rules

- if a link fails with `undefined reference to sce*`, the stub is missing — check
  `Sdk/lib` for the exact library name
- if a call resolves at build time but returns `0x8002xxxx` at runtime, the module
  is not loaded — `sceSysmoduleLoadModule` first
- when hooking, always release the hook in `module_stop`, or the next launch
  inherits a corrupted table

## 7. Sources

- `Tools/THGN/src/main.cpp` + `src/import/vita-import*.c`
- `Tools/base_hooker/src/base_hooker.c` + `base_hooker.yml`
- `Sdk/include/vita-import.h`
- `docs/18-taihen-plugin-development.md`
