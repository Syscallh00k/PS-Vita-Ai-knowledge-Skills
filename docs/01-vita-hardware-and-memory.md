# 01 — PS Vita Hardware & Memory Model

What the machine is, and the five numbers that decide whether a port is possible.

---

## 1. SoC

| Component | Spec | Consequence |
|-----------|------|-------------|
| CPU | 4× ARM Cortex-A9, ARMv7-A, 32-bit, 444 MHz default | roughly a tenth of one PS3 PPU thread; per-frame work must be small |
| SIMD | NEON 128-bit, VFPv3 | mandatory for texture decode, math, audio |
| L2 | 2 MB shared across the cluster | cache-friendly layouts beat clever algorithms |
| GPU | PowerVR SGX543MP4+ (4 USSE2 cores), 166 MHz | tile/IMR hybrid; very sensitive to state changes and fill rate |
| GPU API | `SceGxm` low level; GLES2 via `vitaGL`; `vita2d` for 2D | see doc 04 |
| RAM | 512 MB LPDDR2 total, 128 MB reserved as VRAM (CDRAM) | three pools compete: system, heap, GPU |
| Display | 960×544 | not 720p; author UI at native or scale |
| Storage | `ux0:` card, `ur0:` internal, `app0:` inside VPK | never write to `app0:` |

Overclock (PSVshell / taiHEN plugins) raises CPU to 500 MHz and GPU to 222 MHz.
That is the difference between 25 and 35 fps in a heavy scene, and every serious
port assumes it is available.

---

## 2. The five memblock types

There is no demand paging for user code. Every buffer is an explicit
`SceKernelAllocMemBlock`. From `Sdk/include/psp2/kernel/sysmem.h`:

| Macro | Cached | Executable | Use |
|-------|--------|-----------|-----|
| `SCE_KERNEL_MEMBLOCK_TYPE_USER_RW` | yes | no | newlib heap backing |
| `SCE_KERNEL_MEMBLOCK_TYPE_USER_RW_UNCACHE` | no | no | GPU-visible buffers, DMA staging |
| `SCE_KERNEL_MEMBLOCK_TYPE_USER_RX` | yes | yes | code/data from an ELF loader |
| `SCE_KERNEL_MEMBLOCK_TYPE_USER_CDRAM_RW` | — | no | render targets, resident textures |
| `sceKernelAllocMemBlockForVM` | — | after sync | JIT output, code caves |

Two rules that cost weeks:

1. A buffer the GPU reads or writes must be uncached, or explicitly flushed.
2. `sceKernelGetMemBlockBase` returns the CPU address; the GPU needs the offset
   (`sceGxmAddressToOffset`) — mixing the two gives silent corruption.

---

## 3. Heap sizing

newlib's heap size is a global in your main translation unit:

```c
int _newlib_heap_size_user = 256 * 1024 * 1024;

#ifdef USE_SCELIBC_IO
int sceLibcHeapSize = 4 * 1024 * 1024;
#endif
```

`soloader-template` uses 256 MB. Play! uses 164 MB. Melee's port reserves
24 MB MEM1 + 16 MB ARAM separately and grows the heap around them.

Declare it **once**; two definitions across translation units silently pick one.

---

## 4. What a realistic budget looks like

| Pool | Budget |
|------|--------|
| code + rodata | 20–30 MB |
| streamed geometry | 40–60 MB |
| resident textures | ≤ 80 MB |
| audio buffers | 4–8 MB |
| I/O staging | 16–32 MB |

Total resident for a 3D title: roughly 150–250 MB. Anything above that needs a
design change, not a micro-optimisation.

---

## 5. Practical ceilings

| Thing | Real limit |
|-------|-----------|
| Internal render resolution | 480×272 … 960×544 (960×544 native) |
| Simultaneous draw calls | low thousands at 30 fps, far fewer with state changes |
| Software-mixed audio voices | 64 at 32 kHz is proven (Melee) |
| VPK size before installs get painful | ~500 MB |
| Comfortable card payload | ~3.5 GB |

---

## 6. Sources

- `Sdk/include/psp2/kernel/sysmem.h` — memblock types
- `Other Ports/melee-Vita-vita-port/platforms/vita/game/vita_platform.h` — MEM1/ARAM sizes
- `Other Emulators/play-master/Source/ui_vita/main.cpp` — 164 MB heap, VM block
- `soloader-template/source/main.c` — 256 MB heap, sceLibcHeapSize
