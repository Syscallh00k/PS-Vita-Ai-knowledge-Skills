---
name: rwx-jit
description: Get executable memory the safe way (VM blocks), or the patching way (kubridge), and avoid the two bugs that make generated code fail randomly.
when: Generating code at runtime, loading a foreign module, or hooking a function.
---

# Skill: RWX / JIT Memory

## Step 1 — Choose the mechanism

| You control the memory | You patch someone else's code |
|------------------------|-------------------------------|
| `sceKernelAllocMemBlockForVM` | `kuKernelCpuUnrestrictedMemcpy` |
| no plugin needed | `kubridge.skprx` required |
| no `UNSAFE` SELF | `UNSAFE` SELF required |
| Play!'s JIT, dynamic recompilers, script VMs | `.so` loaders, hookers |

Default to the VM block. Reach for kubridge only when you must modify an existing
code page.

## Step 2 — VM block pattern

```c
#define RWX_BLOCK_SIZE (16 * 1024 * 1024)

static SceUID g_rwx_uid = -1;
static void* g_rwx_base = NULL;
static uint32_t g_rwx_head = 0;

auto VitaRwxCreate() noexcept -> void {
    g_rwx_uid = sceKernelAllocMemBlockForVM("rwx", RWX_BLOCK_SIZE);
    sceKernelGetMemBlockBase(g_rwx_uid, &g_rwx_base);
}

[[nodiscard]] auto VitaRwxAlloc(uint32_t size) noexcept -> void* {
    size = (size + 15u) & ~15u;                 /* keep alignment */
    void* p = static_cast<char*>(g_rwx_base) + g_rwx_head;
    g_rwx_head += size;
    return p;
}

[[nodiscard]] auto VitaRwxSync(void* addr, uint32_t size) noexcept -> int {
    return sceKernelSyncVMDomain(g_rwx_uid, addr, size);
}
```

## Step 3 — The two bugs

**Bug 1: forgot the sync.** Write instructions, never call
`sceKernelSyncVMDomain` → the CPU executes stale or unmapped instructions. Symptom:
crashes that look random and reproduce "sometimes".

**Bug 2: forgot the I-cache flush.** After `kuKernelCpuUnrestrictedMemcpy` into code
you must call `kuKernelFlushCaches`. ARM has separate I- and D-caches; writing code
without invalidating I-cache is exactly as broken as it sounds.

## Step 4 — Hooking encoding

```
ARM:    LDR  PC, [PC, #-4]   0xE51FF004     .word <dst>
THUMB:  LDR.W PC, [PC]       0xF8DF F000    .word <dst>
```

Call-through = restore the original words, flush, call, re-patch, flush. That is
`SO_CONTINUE` in `so_util`, and it is worth copying rather than reinventing.

## Step 5 — Range limits

ARM `B` reaches ±32 MB. Any trampoline must live near the code it replaces. Allocate
with a range check and fail loudly instead of silently branching into nowhere.

## Step 6 — Validate before and after

- before patching: check the target address is inside a module you expect
  (`sceKernelGetModuleInfoForKernel` or `taiGetModuleInfo`)
- after allocating: `sceKernelGetMemBlockBase` must be non-NULL
- after writing: sync/flush, then execute one known-good instruction sequence as a
  self-test before running anything real

## Step 7 — Teardown

Free the block on exit, and release every hook. A relaunch inherits nothing, so
leaked blocks quietly lower the ceiling on the next boot.

## Never

- never execute freshly written data without sync **and** flush
- never hardcode a destination address without `noASLR` or a range check
- never patch an offset you have not verified with a byte signature
