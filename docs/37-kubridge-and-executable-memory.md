# 37 — kubridge & Executable Memory

The plugin that lets you write into code pages, and the version/firmware matrix that
decides whether your port starts on someone else's console.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/runtime.md` §4b–4c, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Two ways to get executable memory

| Approach | Needs kubridge | Use |
|----------|----------------|-----|
| `sceKernelAllocMemBlockForVM` + `sceKernelSyncVMDomain` | no | your own JIT/dynarec (doc 07) |
| `kuKernelAllocMemBlock` with `USER_RX` + `kuKernelCpuUnrestrictedMemcpy` | yes | loading foreign modules, hooking, code caves |

A dynarec does **not** have to depend on kubridge. DaedalusX64-style dynarecs
allocate VM memory and sync it — which is why they run on stock homebrew setups.

## 2. Version matrix

| Release | Adds |
|---------|------|
| v0.1 (TheOfficialFloW, 2021-01) | `kuKernelAllocMemBlock`, `kuKernelFlushCaches`, `kuKernelCpuUnrestrictedMemcpy` |
| v0.2 (bythos14, 2022-08) | `kuKernelMemProtect`, `MemReserve`/`MemCommit`/`MemDecommit`, abort handler |
| v0.3 (2022-10) | `kuKernelRegisterExceptionHandler` / release |
| v0.3.1, **v0.3.1_hotfix** (2024-02/03) | clock get/set, `SWP` instruction enabled on all cores, deadlock fix |

Older so-loader READMEs link v0.1, which has none of the later calls. If you call
`kuKernelMemProtect` or the exception handlers, **name the required version in your
README** and pin the hotfix.

## 3. Firmware limit — the one that bites users

`MemProtect`, `MemReserve`/`Commit`/`Decommit` and the RWX page-table patch use
hard-coded SceSysmem offsets for **3.60 and 3.65 only**. On other firmware kubridge
logs "Offsets are unknown for this firmware. Crashes may occur" and reuses the 3.65
offsets — untested elsewhere.

`AllocMemBlock`, `FlushCaches`, `CpuUnrestrictedMemcpy` and the exception handlers
resolve by NID and have no such limit.

So: anything built on `MemProtect` must state its supported firmware.

## 4. Start without the plugin, then say what is missing

Link the **weak** stub and check at boot:

```c
#include <psp2/vshbridge.h>            /* link -lSceVshBridge_stub */

static int g_probe[2] = { 0, 0 };

[[nodiscard]] auto KuBridgeLoaded() noexcept -> bool {
    return _vshKernelSearchModuleByName("kubridge", g_probe) >= 0;
}
```

Then fail loudly with an on-screen message instead of crashing three frames later.

A subtler variant: with the weak stub, an import the installed kubridge does not
export stays unresolved. Detect it by checking that the stub's third word no longer
holds the function NID (SceLibcPosix does exactly this in
`src/internal/vitamman.c`), or gate on the version note in your README.

## 5. Fixed virtual addresses

`kuKernelAllocMemBlock` with `opt.attr = 0x1` (HAS_VBASE) and `opt.field_C = vaddr`
places a block at a chosen address. DSVita puts its JIT block at `0x98000000` and
guest memory at fixed addresses this way; Android `.so` loaders map each library at
a distinct base (`0x98000000`, `0x96000000`, `0x94000000`, …) so they cannot overlap.

Fixed addresses are what make `noASLR` (doc 18) meaningful — and what make hardcoded
offsets a liability.

## 6. Reserve then commit on demand

```
kuKernelMemReserve(&addr, size, type);      // address space, no pages
kuKernelMemCommit(addr, len, prot, opt);    // back part of it
kuKernelMemDecommit(addr, len);             // release
```

With `KU_KERNEL_MEM_COMMIT_ATTR_HAS_BASE` plus `baseBlock`/`baseOffset` you can commit
another block's pages, producing **mirrored views of one memory** — the trick used for
emulated memory mirrors in DS emulation.

## 7. What so-loaders actually need from kubridge

| Call | Purpose |
|------|---------|
| `kuKernelCpuUnrestrictedMemcpy` | write into read-only `.text` (hooks) |
| `kuKernelAllocMemBlock` + `USER_RX` (0x0C20D050) | trampolines (the type is not in the user headers; loaders define it themselves) |
| `kuKernelFlushCaches` | **after every code write** |
| exception handlers | register dumps without a core dump (doc 35) |

Old ARM `.so` files that use the `SWP` instruction need the v0.3.1 hotfix build.

## 8. Attribution note

SonicMastr/SceLibcPosix implements POSIX `mmap`/`mprotect` on kubridge — read it for
the design. It ships no licence file, so read it as a reference, do not copy it.
