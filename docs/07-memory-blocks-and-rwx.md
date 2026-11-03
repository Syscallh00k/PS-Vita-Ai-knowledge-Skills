# 07 — Memory Blocks & RWX

Everything about executable memory, JIT arenas, and patching code at runtime.

---

## 1. Allocating a block

```c
SceUID block = sceKernelAllocMemBlock("name",
        SCE_KERNEL_MEMBLOCK_TYPE_USER_RW_UNCACHE, size, NULL);
void* base = NULL;
sceKernelGetMemBlockBase(block, &base);
/* ... */
sceKernelFreeMemBlock(block);
```

`base` is the **CPU address**. The GPU needs `sceGxmAddressToOffset(base, &offset)`
before it can sample or render into it. Confusing the two produces silent garbage.

## 2. VM blocks — the safe way to get RWX

`sceKernelAllocMemBlockForVM` gives you memory you can write, then map executable.
Play! uses a single 16 MB arena:

```c
#define RWX_BLOCK_SIZE (16 * 1024 * 1024)

static SceUID g_rwx_uid = -1;
static void* g_rwx_base = NULL;
static uint32_t g_rwx_head = 0;

auto VitaRwxBlockCreate() noexcept -> void {
    g_rwx_uid = sceKernelAllocMemBlockForVM("Play_Vita_rwx_block", RWX_BLOCK_SIZE);
    sceKernelGetMemBlockBase(g_rwx_uid, &g_rwx_base);
}

[[nodiscard]] auto VitaRwxAlloc(uint32_t size) noexcept -> void* {
    void* head = static_cast<char*>(g_rwx_base) + g_rwx_head;
    g_rwx_head += size;
    return head;
}

[[nodiscard]] auto VitaRwxSync(void* addr, uint32_t size) noexcept -> int {
    return sceKernelSyncVMDomain(g_rwx_uid, addr, size);
}
```

`sceKernelSyncVMDomain` is **mandatory** after writing generated code. Skip it and
the CPU executes stale instructions — a bug that appears "randomly" and is
impossible to debug from symptoms.

Because the memory is yours, this path needs **no kubridge** and no `UNSAFE` SELF.

## 3. Patching someone else's code — kubridge

When you must modify code pages you do not own (loading an Android `.so`, hooking
a function), you need kernel help:

```c
kuKernelCpuUnrestrictedMemcpy(dst, src, size);
kuKernelFlushCaches(dst, size);
```

`kuKernelCpuUnrestrictedMemcpy` bypasses page protection. The flush is not optional
— ARM has separate I-cache and D-cache, and modifying code without invalidating the
I-cache is exactly as broken as it sounds.

Requires `kubridge.skprx` installed and a `UNSAFE` SELF. Always check at boot:

```c
if (!module_loaded("kubridge")) {
    FatalError("kubridge.skprx is not installed.");
}
```

## 4. Hook encoding (from `so_util`)

```
ARM:    LDR  PC, [PC, #-4]     0xE51FF004
        .word <destination>

THUMB:  LDR.W PC, [PC]         0xF8DF F000
        .word <destination>
```

Call-through restores the original two words, calls, then re-patches:

```c
#define SO_CONTINUE(type, h, ...) ({ \
  kuKernelCpuUnrestrictedMemcpy((void *)h.addr, h.orig_instr, sizeof(h.orig_instr)); \
  kuKernelFlushCaches((void *)h.addr, sizeof(h.orig_instr)); \
  type r = h.thumb_addr ? ((type(*)())h.thumb_addr)(__VA_ARGS__) \
                        : ((type(*)())h.addr)(__VA_ARGS__); \
  kuKernelCpuUnrestrictedMemcpy((void *)h.addr, h.patch_instr, sizeof(h.patch_instr)); \
  kuKernelFlushCaches((void *)h.addr, sizeof(h.patch_instr)); \
  r; })
```

## 5. Range constraints

ARM `B` reaches ±32 MB. Trampolines and code caves must therefore be allocated
**near** the code they patch. `so_util`'s `so_alloc_arena()` takes a `range`
argument and refuses an allocation that cannot be branched to — copy that discipline
instead of discovering it in a crash.

## 6. Fixed addresses

`Tools/noASLR` disables userland ASLR via a kernel hook so hardcoded addresses stay
valid across runs. Legitimate uses: RE tooling, replaying a specific memory layout,
matching a guide. It also documents why hardcoding addresses is fragile by default.

## 7. Leak discipline

- every `sceKernelAllocMemBlock` needs a matching free on shutdown
- a relaunch (`sceAppMgrLoadExec`) inherits nothing but also frees nothing —
  leaked blocks reduce the ceiling on the next boot
- assert in dev builds: track block UIDs, print any still alive at exit

## 8. Sources

- `Other Emulators/play-master/Source/ui_vita/main.cpp` — VM block
- `Other Ports/raider-vita-master/loader/so_util.c` — hooking, arenas
- `soloader-template/source/patch.c` — kuser cmpxchg/memory-barrier patching
- `Tools/noASLR/` — ASLR disable
- `Sdk/include/psp2/kernel/sysmem.h`
