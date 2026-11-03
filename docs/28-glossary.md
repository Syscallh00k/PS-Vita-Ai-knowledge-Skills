# 28 — Glossary

Terms used across these documents, in the order you will meet them.

---

## Platform

| Term | Meaning |
|------|---------|
| **SELF** | signed ELF — the container the Vita executes; `.self`, and inside a VPK it is `eboot.bin` |
| **VPK** | zip with `sce_sys/` + `eboot.bin` at root; the installable package |
| **bubble** | the home-screen icon for an app |
| **LiveArea** | the launch screen of a bubble: background, startup image, links |
| **`app0:`** | the mounted VPK, read-only |
| **`ux0:`** | memory card / SD2Vita |
| **`ur0:`** | internal storage (system) |
| **`vs0:`** | system partition, read-only for apps |
| **HENkaku / taiHEN** | homebrew enabler / the hooking framework plugins use |
| **skprx / suprx** | kernel module / userland module |
| **kubridge** | kernel plugin giving `kuKernel*` calls (unrestricted memcpy, cache flush) |
| **PSVshell** | overclock plugin (500 MHz CPU). Not required; never depend on it |

## Graphics

| Term | Meaning |
|------|---------|
| **GXM** | Sony's low-level GPU API on Vita (`sceGxm*`) |
| **SGX543MP4+** | the GPU: PowerVR, 4 USSE2 cores |
| **shader patcher** | `SceGxmShaderPatcher` — builds programs from `.gxp` + stream/blend config |
| **`.gxp`** | compiled shader program container |
| **`psp2cgc`** | offline Cg compiler producing `.gxp` |
| **`libshacccg.suprx`** | on-device shader compiler module (user extracts it themselves) |
| **vitaShaRK** | wrapper over the on-device Cg compiler |
| **vitaGL** | GLES2 → GXM translation layer |
| **vita2d** | simple 2D/texture API over GXM |
| **render target** | offscreen surface; MSAA sample count fixed at creation |
| **swizzle** | Morton-order texel layout the SGX prefers |
| **EFB** | embedded frame buffer (GameCube term; appears in port code) |
| **TEV** | GameCube texture environment unit; appears as "TEV stage" in GX ports |
| **RSX** | PS3 GPU — shaders cooked as microcode, not source |

## Memory

| Term | Meaning |
|------|---------|
| **memblock** | `sceKernelAllocMemBlock` allocation; the only way to get memory |
| **CDRAM** | the 128 MB VRAM pool |
| **VM block** | `sceKernelAllocMemBlockForVM` — writable then executable memory |
| **RWX** | read/write/execute memory |
| **`_newlib_heap_size_user`** | the global that sizes newlib's heap |
| **`sceKernelSyncVMDomain`** | mandatory cache sync after writing generated code |

## Porting

| Term | Meaning |
|------|---------|
| **decompilation** | source recovered from a binary, matched to the original |
| **platform layer** | the new code that replaces the console SDK |
| **NID** | 32-bit hash identifying an exported function |
| **stub** | link library containing NIDs, resolved at load time |
| **so_util** | the ELF loader used to run Android `.so` files |
| **FalsoJNI / FalsoNDK** | fake Java VM / fake NDK activity layer |
| **FIOS overlay** | runtime path remapping (`/data/...` → `ux0:data/game/...`) |
| **kuser helpers** | Android fixed-address `__kuser_cmpxchg` / `__kuser_memory_barrier` |
| **dynarec** | dynamic recompiler (how an emulator runs guest CPU code fast) |
| **PCK / UPK** | Godot package / Unreal Engine 3 package |
| **TFC** | UE3 texture file cache (shared texture blob) |
| **FSB** | FMOD sample bank |
| **Bink (`.bik`)** | RAD game video format; no Vita decoder exists |

## Measurement

| Term | Meaning |
|------|---------|
| **frame budget** | 16.6 ms at 60 fps, 33.3 ms at 30 fps |
| **internal resolution** | resolution actually rendered, upscaled to 960×544 |
| **overdraw** | how many times a pixel is shaded in one frame |
| **shader permutation** | one compiled variant of a shader for a specific state combination |
| **warm cache** | a pre-trained shader cache shipped with the build |

---

Back to the [README](../README.md).
