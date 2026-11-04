# PS-Vita-Ai-knowledge-Skills | Gathered using deepseek

Notes, references and task playbooks assembled from real, working PS Vita projects:
a game port compiled from a decompilation, an Android `.so` loader, a PS1 emulator,
a PS2 emulator frontend, a Godot content patch, a 25-tool dump of Vita development
utilities, and a bare VitaSDK tree.

---

## Documentation

| # | Document | What it covers |
|---|----------|----------------|
| 01 | [Hardware & Memory Model](docs/01-vita-hardware-and-memory.md) | SoC, CPU/GPU, RAM split, memblock types, heap sizing |
| 02 | [VitaSDK Reference](docs/02-vitasdk-reference.md) | What the SDK tree contains, module headers, stub libs, what's missing |
| 03 | [Build & Packaging](docs/03-build-and-packaging.md) | CMake, `vita.cmake`, SELF/VPK, LiveArea, deploy loop |
| 04 | [GXM Rendering](docs/04-gxm-rendering.md) | Contexts, shader patcher, render targets, state, blend, depth |
| 05 | [Shader Toolchain](docs/05-shader-toolchain.md) | `psp2cgc`, `.gxp`, `libshacccg`, vitaShaRK, GLSL via vitaGL |
| 06 | [Filesystem & I/O](docs/06-filesystem-and-io.md) | `ux0:`/`ur0:`/`app0:`, FIOS, async I/O, streaming threads |
| 07 | [Memory Blocks & RWX](docs/07-memory-blocks-and-rwx.md) | `sceKernelAllocMemBlock`, VM blocks, JIT arenas, kubridge |
| 08 | [Threading & Timing](docs/08-threading-and-timing.md) | pthread vs kernel threads, priorities, clocks, pacing |
| 09 | [Input, HID & Motion](docs/09-input-hid-and-motion.md) | `SceCtrl`, `SceTouch`, `SceMotion`, remapping |
| 10 | [Audio Output](docs/10-audio-output.md) | `SceAudioOut`, `SceAudioIn`, `SceAtrac`, ATRAC9, mixing |
| 11 | [Video & Media](docs/11-video-and-media.md) | `SceAvPlayer`, H.264 constraints, transcoding |
| 12 | [LiveArea & param.sfo](docs/12-livearea-and-param-sfo.md) | Bubble metadata, attribute flags, templates, icons |
| 13 | [Trophies & NPDRM](docs/13-trophies-and-npdrm.md) | `SceNpTrophy`, trophy packs, `SceNpDrm` stubs |
| 14 | [Debugging & Logging](docs/14-debugging-and-logging.md) | File logs, UDP loggers, on-screen debug screens, log plugins |
| 15 | [Crash & Error Diagnostics](docs/15-crash-and-error-diagnostics.md) | `0x8xxxxxxx` codes, kernel dumps, physmem dumps, core parsing |
| 16 | [SELF & ELF Format](docs/16-self-and-elf-format.md) | Vita SELF structure, `sceModuleInfo`, ELF↔SELF conversion |
| 17 | [NID & Import System](docs/17-nid-and-import-system.md) | NIDs, module imports/exports, YAML stubs, name resolution |
| 18 | [taiHEN Plugin Development](docs/18-taihen-plugin-development.md) | skprx/suprx modules, hooking, `exports.yml`, real examples |
| 19 | [Porting Android `.so` Games](docs/19-porting-android-so.md) | so_util, FalsoJNI, libc_bridge, FIOS, vitaGL, patches |
| 20 | [Porting Decompiled Source](docs/20-porting-decompiled-source.md) | The Melee pattern: platform layer swap, ABI flags, big-endian |
| 21 | [Porting Emulator Frontends](docs/21-porting-emulator-frontends.md) | PCSX AwO and Play!: cores, dynarec, GXM backends, config |
| 22 | [Porting by Data Patching](docs/22-porting-data-patching.md) | Godot PCK: extract, diff, repack, package |
| 23 | [Performance & Budgets](docs/23-performance-and-budgets.md) | Frame budget, memory budget, instrumenting, resolution scaling |
| 24 | [Asset Conversion](docs/24-asset-conversion.md) | Textures, swizzle, mips, audio, video, packed archives |
| 25 | [Testing with Vita3K](docs/25-testing-with-vita3k.md) | What the emulator catches, what it lies about, setup flags |
| 26 | [PS3 / UE3 Target Notes](docs/26-ps3-and-ue3-target-notes.md) | PPC64 SELF, cooked packages, RSX shader notes, size math |
| 27 | [Provenance & Distribution](docs/27-provenance-and-distribution.md) | Shipping tools not data, patching user copies, licensing |
| 28 | [Glossary](docs/28-glossary.md) | Every term used here, in the order you meet it |
| 29 | [vitaGL](docs/29-vitagl.md) | GL → GXM: stale names, limits, the 49152-vertex cap, pool sizing |
| 30 | [Static Recompilation](docs/30-static-recompilation.md) | Lifting a closed binary to C; host-first ordering; shader-bank strategy |
| 31 | [Cross-Platform API Emulation](docs/31-api-emulation-hal.md) | Porting someone else's platform API instead of the game |
| 32 | [Porting Apps & Tools](docs/32-porting-apps-and-tools.md) | Non-game targets: IME, dialogs, permissions, media stack |
| 33 | [Reference Projects Catalogue](docs/33-reference-projects.md) | Who already shipped what, and which patterns to read |
| 34 | [Build/Install Error Catalogue](docs/34-build-install-error-catalogue.md) | `0x8010113D` and friends, plus the linker errors |
| 35 | [Static Fault Analysis](docs/35-static-fault-analysis.md) | Address → named defect, `--wrap=abort`, null-deref signature |
| 36 | [32-bit Portability Audit](docs/36-32bit-portability-audit.md) | Enum split-build, alignment, the hardware-check trap |
| 37 | [kubridge & Executable Memory](docs/37-kubridge-and-executable-memory.md) | Version matrix, 3.60/3.65 limits, weak-stub boot check |
| 38 | [Java/libGDX via TeaVM](docs/38-java-games-teavm.md) | AOT route for Java games, and when it does not fit |
| 39 | [Tooling Scripts](docs/39-tooling-scripts.md) | The four validators and three templates shipped here |
| 40 | [Runtime Shader Compiler](docs/40-runtime-shader-compiler.md) | vitaShaRK: init, options, logging, patcher chain, caching |
| 41 | [Kernel Debugging with GDB](docs/41-kernel-debugging-gdb.md) | kvdb, UART/TCP, what it can't do, ELF discipline |
| 42 | [A Complete Direct-GXM Sample](docs/42-direct-gxm-sample.md) | `gxmfun` read end to end: stride, pipe sync, embedded `.gxp` |
| 43 | [GameMaker: YoYo Loader](docs/43-gamemaker-yoyo-loader.md) | Runner loading, softfp toolchain, asset swap, PVRTC2 |
| 44 | [Emulator Port Anatomy](docs/44-emulator-port-anatomy.md) | Platform directory, dynarec vs interpreter, memory, frontend |
| 45 | [id Tech Ports](docs/45-idtech-quake-port.md) | Quake II / Wolf3D: a `psp2/` directory as the whole port |
| 46 | [Homebrew Engines & Unity](docs/46-engines-xenity-and-unity.md) | Xenity's shared core; the Unity Vita reality |
| 47 | [Engine as a Shared Library](docs/47-engine-as-shared-library.md) | `libgodot.so` + import-resolving loader; TLS shims; package-time import checks |
| 48 | [pthread & libstdc++ Traps](docs/48-pthread-and-libstdcpp-traps.md) | The `pthread_cancel` probe: instant crash *and* silent heap corruption |
| 49 | [Networking & TLS](docs/49-networking-and-tls.md) | Why SceHttp/SceSsl fail, and the libcurl+mbedTLS policy that works |
| 50 | [Interpreter & Data-Driven Engines](docs/50-interpreter-and-data-driven-engines.md) | EasyRPG, GemRB, fake-08, Aleph One, FLARE — ship the engine, user brings content |
| 51 | [Scripting Runtimes](docs/51-scripting-runtimes.md) | CPython 2.7 configure recipe; Mono AOT for C# |
| 52 | [Building Existing Projects](docs/52-building-existing-projects.md) | `Makefile.vita`, CMake `-C` presets, what every port disables |



## Skills (task playbooks)

| Skill | Use when |
|-------|----------|
| [setup-vitasdk](skills/skill-setup-vitasdk.md) | First machine setup, or "toolchain not found" |
| [build-vpk](skills/skill-build-vpk.md) | Turning a project into an installable bubble |
| [first-frame](skills/skill-first-frame.md) | Nothing renders; bring up display → shader → texture |
| [shader-cache](skills/skill-shader-cache.md) | Shader stutter, or runtime compilation is required |
| [rwx-jit](skills/skill-rwx-jit.md) | Generating or patching executable memory |
| [streaming-io](skills/skill-streaming-io.md) | Loading large assets without hitching |
| [audio-mixer](skills/skill-audio-mixer.md) | Getting sound out, or fixing crackle |
| [debug-logging](skills/skill-debug-logging.md) | No PC attached, or you need a live log tail |
| [crash-triage](skills/skill-crash-triage.md) | App dies, hangs, or returns a weird code |
| [self-to-elf](skills/skill-self-to-elf.md) | Analysing a SELF/ELF container |
| [nid-mapping](skills/skill-nid-mapping.md) | Unknown imports, unknown exports, hooking by NID |
| [taihen-plugin](skills/skill-taihen-plugin.md) | Writing a kernel/user plugin to patch a running game |
| [port-triage](skills/skill-port-triage.md) | Deciding which port family applies, before writing code |
| [android-so-port](skills/skill-android-so-port.md) | Porting an Android ARMv7 game binary |
| [decomp-port](skills/skill-decomp-port.md) | Porting a game you have source or a decompilation for |
| [data-patch-port](skills/skill-data-patch-port.md) | Repacking content for an engine that already runs on Vita |
| [vitagl-port](skills/skill-vitagl-port.md) | The renderer calls `gl*` and needs to land on vitaGL |
| [static-recomp](skills/skill-static-recomp.md) | Lifting a closed-source binary instead of porting source |
| [app-port](skills/skill-app-port.md) | The target is a utility, media player or CLI tool |
| [vpk-validate](skills/skill-vpk-validate.md) | Installs but no bubble, or a console error code |
| [shader-runtime-compile](skills/skill-shader-runtime-compile.md) | Compiling shaders on device with vitaShaRK |
| [gdb-debug](skills/skill-gdb-debug.md) | Attaching a real debugger with kvdb |
| [emulator-port](skills/skill-emulator-port.md) | Porting an emulator, or a game only reachable by emulation |
| [gamemaker-port](skills/skill-gamemaker-port.md) | Running a GameMaker title via YoYo Loader |
| [pthread-cpp-crash](skills/skill-pthread-cpp-crash.md) | C++ app dies on its first `std::thread`, or corrupts the heap |
| [https-downloads](skills/skill-https-downloads.md) | Adding working HTTPS downloads on device |
| [engine-as-so](skills/skill-engine-as-so.md) | Splitting an engine into a `.so` + loader |
| [release-pipeline](skills/skill-release-pipeline.md) | Versioning, CI, signed updates, shipping |


## Conventions

- One house style for all code: PascalCase functions, `auto Fn() -> T` trailing
  returns, `g_` globals, snake_case locals, `#ifndef` include guards.
- Every constant quoted (clocks, block sizes, limits) comes from a header in
  `Sdk/include/psp2` or from a working project listed above.
- "Never" lines are mistakes that have already cost someone a week.

## Start here

| If you are… | Read first |
|-------------|-----------|
| setting up a machine | [setup-vitasdk](skills/skill-setup-vitasdk.md) → [03 Build & Packaging](docs/03-build-and-packaging.md) |
| deciding whether a port is possible | [port-triage](skills/skill-port-triage.md) → [26 Target Notes](docs/26-ps3-and-ue3-target-notes.md) |
| stuck on a black screen | [first-frame](skills/skill-first-frame.md) → [04 GXM Rendering](docs/04-gxm-rendering.md) |
| chasing a crash | [crash-triage](skills/skill-crash-triage.md) → [15 Diagnostics](docs/15-crash-and-error-diagnostics.md) |
| porting an Android game | [android-so-port](skills/skill-android-so-port.md) → [19 Android `.so`](docs/19-porting-android-so.md) |
| porting a decompilation | [decomp-port](skills/skill-decomp-port.md) → [20 Decompiled Source](docs/20-porting-decompiled-source.md) |
| shipping | [release-pipeline](skills/skill-release-pipeline.md) → [27 Provenance](docs/27-provenance-and-distribution.md) |


